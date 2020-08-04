# 表达式预处理源码分析

在完成FROM...WHERE子句的优化后，需要对targelist以及条件子句中表达式进行优化。
## 处理targetlist子句

可以看到处理targetlist子句调用是preprocess_expression函数，其实其他子句中的表达式大多也是该函数处理。的。

```c
parse->targetList = (List *)
		preprocess_expression(root, (Node *) parse->targetList,
							  EXPRKIND_TARGET);
```

## preprocess_expression函数处理流程

```c
/*
 * preprocess_expression
 *		Do subquery_planner's preprocessing work for an expression,
 *		which can be a targetlist, a WHERE clause (including JOIN/ON
 *		conditions), a HAVING clause, or a few other things.
 */
static Node *
preprocess_expression(PlannerInfo *root, Node *expr, int kind)
{
	/*
	 * Fall out quickly if expression is empty.  This occurs often enough to
	 * be worth checking.  Note that null->null is the correct conversion for
	 * implicit-AND result format, too.
	 */
	if (expr == NULL)
		return NULL;

	/*
	 * If the query has any join RTEs, replace join alias variables with
	 * base-relation variables.  We must do this first, since any expressions
	 * we may extract from the joinaliasvars lists have not been preprocessed.
	 * For example, if we did this after sublink processing, sublinks expanded
	 * out from join aliases would not get processed.  But we can skip this in
	 * non-lateral RTE functions, VALUES lists, and TABLESAMPLE clauses, since
	 * they can't contain any Vars of the current query level.
	 */
	if (root->hasJoinRTEs &&
		!(kind == EXPRKIND_RTFUNC ||
		  kind == EXPRKIND_VALUES ||
		  kind == EXPRKIND_TABLESAMPLE ||
		  kind == EXPRKIND_TABLEFUNC))
		expr = flatten_join_alias_vars(root->parse, expr);

	/*
	 * Simplify constant expressions.
	 *
	 * Note: an essential effect of this is to convert named-argument function
	 * calls to positional notation and insert the current actual values of
	 * any default arguments for functions.  To ensure that happens, we *must*
	 * process all expressions here.  Previous PG versions sometimes skipped
	 * const-simplification if it didn't seem worth the trouble, but we can't
	 * do that anymore.
	 *
	 * Note: this also flattens nested AND and OR expressions into N-argument
	 * form.  All processing of a qual expression after this point must be
	 * careful to maintain AND/OR flatness --- that is, do not generate a tree
	 * with AND directly under AND, nor OR directly under OR.
	 */
	expr = eval_const_expressions(root, expr);

	/*
	 * If it's a qual or havingQual, canonicalize it.
	 */
	if (kind == EXPRKIND_QUAL)
	{
		expr = (Node *) canonicalize_qual((Expr *) expr, false);

#ifdef OPTIMIZER_DEBUG
		printf("After canonicalize_qual()\n");
		pprint(expr);
#endif
	}

	/* Expand SubLinks to SubPlans */
	if (root->parse->hasSubLinks)
		expr = SS_process_sublinks(root, expr, (kind == EXPRKIND_QUAL));

	/*
	 * XXX do not insert anything here unless you have grokked the comments in
	 * SS_replace_correlation_vars ...
	 */

	/* Replace uplevel vars with Param nodes (this IS possible in VALUES) */
	if (root->query_level > 1)
		expr = SS_replace_correlation_vars(root, expr);

	/*
	 * If it's a qual or havingQual, convert it to implicit-AND format. (We
	 * don't want to do this before eval_const_expressions, since the latter
	 * would be unable to simplify a top-level AND correctly. Also,
	 * SS_process_sublinks expects explicit-AND format.)
	 */
	if (kind == EXPRKIND_QUAL)
		expr = (Node *) make_ands_implicit((Expr *) expr);

	return expr;
}
```

下面给出函数执行的流程:

- 如果该查询含有多表连接的RTE，需要将别名转换成基表信息(flatten_join_alias_vars)
- 简化常量表达式(eval_const_expressions)
- 规范条件子句或者having子句中的表达式(canonicalize_qual)
- 将子链接转换为子查询计划(SS_process_sublinks)，注意这里的子链接指的是没有被优化的子链接。
- 将上层的VAR变量转换成Param节点（SS_replace_correlation_vars）
	 把where和having中的and表达式转换为隐含形式 	

## 处理条件子句

从上面可以看出对于查询中各类子句表达式，都是先调用precess_expression进行预处理。所以这里我们重点讨论条件子句中表达式的执行。
```c
//预处理条件子句
preprocess_qual_conditions(root, (Node *) parse->jointree);

/*
 * preprocess_qual_conditions
 *		Recursively scan the query's jointree and do subquery_planner's
 *		preprocessing work on each qual condition found therein.
 */
static void
preprocess_qual_conditions(PlannerInfo *root, Node *jtnode)
{
	if (jtnode == NULL)
		return;
	if (IsA(jtnode, RangeTblRef))
	{
		/* nothing to do here */
	}
	else if (IsA(jtnode, FromExpr))
	{
		FromExpr   *f = (FromExpr *) jtnode;
		ListCell   *l;

		foreach(l, f->fromlist)
			preprocess_qual_conditions(root, lfirst(l));

		f->quals = preprocess_expression(root, f->quals, EXPRKIND_QUAL);
	}
	else if (IsA(jtnode, JoinExpr))
	{
		JoinExpr   *j = (JoinExpr *) jtnode;

		preprocess_qual_conditions(root, j->larg);
		preprocess_qual_conditions(root, j->rarg);

		j->quals = preprocess_expression(root, j->quals, EXPRKIND_QUAL);
	}
	else
		elog(ERROR, "unrecognized node type: %d",
			 (int) nodeTag(jtnode));
}
```

先看preprocess_qual_conditions函数的执行流程:

遍历jointree,然后根据节点的类型，递归的调用preprocess_qual_conditions进行分类处理。如果是quals子句则调用preprocess_expression函数去处理。如果该查询包含子链接，则调用SS_process_sublinks函数(实际调用process_sublinks_mutator函数)去处理quals子句，而process_sublinks_mutator函数首先判断quals子句的类型是不是给定的类型，如果是，则根据给定的类型进行处理；如果不是，会调用通用函expression_tree_mutator，完成对该子树的处理。

(这里主要关注quals子句为sublink类型的处理)如果该语句中含有sublink类型则调用SS_process_sublinks函数处理(实际调用process_sublinks_mutator函数)，首先处理sublink中的testexpr,然后将sublink节点通过make_subplan函数转换为SubPlan节点。

#### SS_process_sublinks

这里主要讨论前面在优化时没有被优化的子链接该如何处理？

该函数主要的作用是处理FROM...WHERE子句中没有被优化的子链接。

```c
/*
 * Expand SubLinks to SubPlans in the given expression.
 *
 * The isQual argument tells whether or not this expression is a WHERE/HAVING
 * qualifier expression.  If it is, any sublinks appearing at top level need
 * not distinguish FALSE from UNKNOWN return values.
 */
Node *
SS_process_sublinks(PlannerInfo *root, Node *expr, bool isQual)
{
	process_sublinks_context context;

	context.root = root;
	context.isTopQual = isQual;
	return process_sublinks_mutator(expr, &context);
}

static Node *
process_sublinks_mutator(Node *node, process_sublinks_context *context)
{
	process_sublinks_context locContext;

	locContext.root = context->root;

	if (node == NULL)
		return NULL;
	if (IsA(node, SubLink))
	{
		SubLink    *sublink = (SubLink *) node;
		Node	   *testexpr;

		/*
		 * First, recursively process the lefthand-side expressions, if any.
		 * They're not top-level anymore.
		 */
		locContext.isTopQual = false;
		testexpr = process_sublinks_mutator(sublink->testexpr, &locContext);

		/*
		 * Now build the SubPlan node and make the expr to return.
		 */
		return make_subplan(context->root,
							(Query *) sublink->subselect,
							sublink->subLinkType,
							sublink->subLinkId,
							testexpr,
							context->isTopQual);
	}

	/*
	 * Don't recurse into the arguments of an outer PHV or aggregate here. Any
	 * SubLinks in the arguments have to be dealt with at the outer query
	 * level; they'll be handled when build_subplan collects the PHV or Aggref
	 * into the arguments to be passed down to the current subplan.
	 */
	if (IsA(node, PlaceHolderVar))
	{
		if (((PlaceHolderVar *) node)->phlevelsup > 0)
			return node;
	}
	else if (IsA(node, Aggref))
	{
		if (((Aggref *) node)->agglevelsup > 0)
			return node;
	}

	/*
	 * We should never see a SubPlan expression in the input (since this is
	 * the very routine that creates 'em to begin with).  We shouldn't find
	 * ourselves invoked directly on a Query, either.
	 */
	Assert(!IsA(node, SubPlan));
	Assert(!IsA(node, AlternativeSubPlan));
	Assert(!IsA(node, Query));

	/*
	 * Because make_subplan() could return an AND or OR clause, we have to
	 * take steps to preserve AND/OR flatness of a qual.  We assume the input
	 * has been AND/OR flattened and so we need no recursion here.
	 *
	 * (Due to the coding here, we will not get called on the List subnodes of
	 * an AND; and the input is *not* yet in implicit-AND format.  So no check
	 * is needed for a bare List.)
	 *
	 * Anywhere within the top-level AND/OR clause structure, we can tell
	 * make_subplan() that NULL and FALSE are interchangeable.  So isTopQual
	 * propagates down in both cases.  (Note that this is unlike the meaning
	 * of "top level qual" used in most other places in Postgres.)
	 */
	if (is_andclause(node))
	{
		List	   *newargs = NIL;
		ListCell   *l;

		/* Still at qual top-level */
		locContext.isTopQual = context->isTopQual;

		foreach(l, ((BoolExpr *) node)->args)
		{
			Node	   *newarg;

			newarg = process_sublinks_mutator(lfirst(l), &locContext);
			if (is_andclause(newarg))
				newargs = list_concat(newargs, ((BoolExpr *) newarg)->args);
			else
				newargs = lappend(newargs, newarg);
		}
		return (Node *) make_andclause(newargs);
	}

	if (is_orclause(node))
	{
		List	   *newargs = NIL;
		ListCell   *l;

		/* Still at qual top-level */
		locContext.isTopQual = context->isTopQual;

		foreach(l, ((BoolExpr *) node)->args)
		{
			Node	   *newarg;

			newarg = process_sublinks_mutator(lfirst(l), &locContext);
			if (is_orclause(newarg))
				newargs = list_concat(newargs, ((BoolExpr *) newarg)->args);
			else
				newargs = lappend(newargs, newarg);
		}
		return (Node *) make_orclause(newargs);
	}

	/*
	 * If we recurse down through anything other than an AND or OR node, we
	 * are definitely not at top qual level anymore.
	 */
	locContext.isTopQual = false;

	return expression_tree_mutator(node,
								   process_sublinks_mutator,
								   (void *) &locContext);
}
```

## 示例SQL
```sql
SELECT
    classno,
    classname,
    avg(score) AS avg_score
FROM
    sc,
    (
        SELECT
            *
        FROM
            class
        WHERE
            class.gno = 'grade one') AS sub
WHERE
    sc.sno IN (
        SELECT
            sno
        FROM
            student
        WHERE
            student.classno = sub.classno)
    AND sc.cno IN (
        SELECT
            course.cno
        FROM
            course
        WHERE
            course.cname = 'computer')
GROUP BY
    classno,
    classname
HAVING
    avg(score) > 60
ORDER BY
    avg_score
```
该SQL对应的查询树的结构(经过子链接、子查询优化后)如下:
![img](http://assets.processon.com/chart_image/5f1913ea1e08533a627fd2e6.png) 

### testexpr处理

对于sublink1的处理，首先从testexpr开始，而testexpr的类型为T_OpExpr。而函数process_sublinks_mutator不对该节点做任何变化，只是对其子节点调用expression_tree_mutator进行转换。其子节点为T_list={sc.cno,student.sno},最后在expression_tree_mutator与process_sublinks_mutator反复调用和生成的新的T_list={(var)sc.cno，(param)student.sno}。也就是说testexpr进行的是变量类型的转换操作。下面是sublink1转换前后的对比图:

#### 处理前的testexpr

![img](http://assets.processon.com/chart_image/5f1913ea1e08533a627fd2e6.png) 

#### 处理后的testexpr

![img](http://assets.processon.com/chart_image/5f1963e4637689168e2fcd31.png) 

### make_subplan

完成testexpr的处理后将会调用make_subplan函数将sublink转换成子查询计划。

make_subplan将子链接的类型设置的tuple_fraction和子链接中的子查询作为参数重新进入subquery_planner中

，与父查询一样也需要对子链接中的子查询经过子链接、子查询的优化过程，最后生成一个子查询计划。关于如何构建查询计划，将在后续继续介绍。