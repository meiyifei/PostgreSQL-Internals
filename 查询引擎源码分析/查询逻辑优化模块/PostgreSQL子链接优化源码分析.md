# 	PostgreSQL子链接优化源码分析

## 一、优化的入口函数

通过pg_analyze_and_rewrite函数分析和重写之后输出的为查询树，但这颗查询树显然不是最优的查询树，需要继续优化。

```c
*
 * exec_simple_query
 *
 * Execute a "simple Query" protocol message.
 /*
static void
exec_simple_query(const char *query_string)
{
  ''''''
    querytree_list = pg_analyze_and_rewrite(parsetree, query_string,NULL, 0, NULL);

	plantree_list = pg_plan_queries(querytree_list,CURSOR_OPT_PARALLEL_OK, NULL);
  ''''''
}

//入口函数为pg_plan_queries
/*
 * Generate plans for a list of already-rewritten queries.
 *
 * For normal optimizable statements, invoke the planner.  For utility
 * statements, just make a wrapper PlannedStmt node.
 *
 * The result is a list of PlannedStmt nodes.
 */
List *
pg_plan_queries(List *querytrees, int cursorOptions, ParamListInfo boundParams)
{
	List	   *stmt_list = NIL;
	ListCell   *query_list;

	foreach(query_list, querytrees)
	{
		Query	   *query = lfirst_node(Query, query_list);
		PlannedStmt *stmt;

        /*
        	对于工具类的语句不需要进一步的优化。非工具类的语句需要调用pg_plan_query函数进行优化
        */
		if (query->commandType == CMD_UTILITY)
		{
			/* Utility commands require no planning. */
			stmt = makeNode(PlannedStmt);
			stmt->commandType = CMD_UTILITY;
			stmt->canSetTag = query->canSetTag;
			stmt->utilityStmt = query->utilityStmt;
			stmt->stmt_location = query->stmt_location;
			stmt->stmt_len = query->stmt_len;
		}
		else
		{
            //该函数又调用了planner函数(优化器的入口函数)
			stmt = pg_plan_query(query, cursorOptions, boundParams);
		}

		stmt_list = lappend(stmt_list, stmt);
	}

	return stmt_list;
}
```

### planner函数

```c
/*****************************************************************************
 *
 *	   Query optimizer entry point
 *
 * To support loadable plugins that monitor or modify planner behavior,
 * we provide a hook variable that lets a plugin get control before and
 * after the standard planning process.  The plugin would normally call
 * standard_planner().
 *
 * Note to plugin authors: standard_planner() scribbles on its Query input,
 * so you'd better copy that data structure if you want to plan more than once.
 *
 *****************************************************************************/
PlannedStmt *
planner(Query *parse, int cursorOptions, ParamListInfo boundParams)
{
	PlannedStmt *result;

	if (planner_hook)
		result = (*planner_hook) (parse, cursorOptions, boundParams);
	else
		result = standard_planner(parse, cursorOptions, boundParams);
	return result;
}

//最终执行优化的操作定位到subquery_planner函数，该函数根据不同子句的类型分类进行优化
```

### subquery_planner函数

```c
PlannerInfo *
subquery_planner(PlannerGlobal *glob, Query *parse,
				 PlannerInfo *parent_root,
				 bool hasRecursion, double tuple_fraction)
{
	PlannerInfo *root;
	List	   *newWithCheckOptions;
	List	   *newHaving;
	bool		hasOuterJoins;
	bool		hasResultRTEs;
	RelOptInfo *final_rel;
	ListCell   *l;

	/* Create a PlannerInfo data structure for this subquery */
	root = makeNode(PlannerInfo);
	root->parse = parse;
	root->glob = glob;
	root->query_level = parent_root ? parent_root->query_level + 1 : 1;
	root->parent_root = parent_root;
	root->plan_params = NIL;
	root->outer_params = NULL;
	root->planner_cxt = CurrentMemoryContext;
	root->init_plans = NIL;
	root->cte_plan_ids = NIL;
	root->multiexpr_params = NIL;
	root->eq_classes = NIL;
	root->append_rel_list = NIL;
	root->rowMarks = NIL;
	memset(root->upper_rels, 0, sizeof(root->upper_rels));
	memset(root->upper_targets, 0, sizeof(root->upper_targets));
	root->processed_tlist = NIL;
	root->grouping_map = NULL;
	root->minmax_aggs = NIL;
	root->qual_security_level = 0;
	root->inhTargetKind = INHKIND_NONE;
	root->hasRecursion = hasRecursion;
	if (hasRecursion)
		root->wt_param_id = assign_special_exec_param(root);
	else
		root->wt_param_id = -1;
	root->non_recursive_path = NULL;
	root->partColsUpdated = false;

	/*
	 * If there is a WITH list, process each WITH query and either convert it
	 * to RTE_SUBQUERY RTE(s) or build an initplan SubPlan structure for it.
	 */
	if (parse->cteList)
		SS_process_ctes(root);

	/*
	 * If the FROM clause is empty, replace it with a dummy RTE_RESULT RTE, so
	 * that we don't need so many special cases to deal with that situation.
	 */
	replace_empty_jointree(parse);

	/*
	 * Look for ANY and EXISTS SubLinks in WHERE and JOIN/ON clauses, and try
	 * to transform them into joins.  Note that this step does not descend
	 * into subqueries; if we pull up any subqueries below, their SubLinks are
	 * processed just before pulling them up.
	 */
	if (parse->hasSubLinks)
		pull_up_sublinks(root);

	/*
	 * Scan the rangetable for set-returning functions, and inline them if
	 * possible (producing subqueries that might get pulled up next).
	 * Recursion issues here are handled in the same way as for SubLinks.
	 */
	inline_set_returning_functions(root);

	/*
	 * Check to see if any subqueries in the jointree can be merged into this
	 * query.
	 */
	pull_up_subqueries(root);

	/*
	 * If this is a simple UNION ALL query, flatten it into an appendrel. We
	 * do this now because it requires applying pull_up_subqueries to the leaf
	 * queries of the UNION ALL, which weren't touched above because they
	 * weren't referenced by the jointree (they will be after we do this).
	 */
	if (parse->setOperations)
		flatten_simple_union_all(root);

	/*
	 * Survey the rangetable to see what kinds of entries are present.  We can
	 * skip some later processing if relevant SQL features are not used; for
	 * example if there are no JOIN RTEs we can avoid the expense of doing
	 * flatten_join_alias_vars().  This must be done after we have finished
	 * adding rangetable entries, of course.  (Note: actually, processing of
	 * inherited or partitioned rels can cause RTEs for their child tables to
	 * get added later; but those must all be RTE_RELATION entries, so they
	 * don't invalidate the conclusions drawn here.)
	 */
	root->hasJoinRTEs = false;
	root->hasLateralRTEs = false;
	hasOuterJoins = false;
	hasResultRTEs = false;
	foreach(l, parse->rtable)
	{
		RangeTblEntry *rte = lfirst_node(RangeTblEntry, l);

		switch (rte->rtekind)
		{
			case RTE_RELATION:
				if (rte->inh)
				{
					/*
					 * Check to see if the relation actually has any children;
					 * if not, clear the inh flag so we can treat it as a
					 * plain base relation.
					 *
					 * Note: this could give a false-positive result, if the
					 * rel once had children but no longer does.  We used to
					 * be able to clear rte->inh later on when we discovered
					 * that, but no more; we have to handle such cases as
					 * full-fledged inheritance.
					 */
					rte->inh = has_subclass(rte->relid);
				}
				break;
			case RTE_JOIN:
				root->hasJoinRTEs = true;
				if (IS_OUTER_JOIN(rte->jointype))
					hasOuterJoins = true;
				break;
			case RTE_RESULT:
				hasResultRTEs = true;
				break;
			default:
				/* No work here for other RTE types */
				break;
		}

		if (rte->lateral)
			root->hasLateralRTEs = true;

		/*
		 * We can also determine the maximum security level required for any
		 * securityQuals now.  Addition of inheritance-child RTEs won't affect
		 * this, because child tables don't have their own securityQuals; see
		 * expand_single_inheritance_child().
		 */
		if (rte->securityQuals)
			root->qual_security_level = Max(root->qual_security_level,
											list_length(rte->securityQuals));
	}

	/*
	 * Preprocess RowMark information.  We need to do this after subquery
	 * pullup, so that all base relations are present.
	 */
	preprocess_rowmarks(root);

	/*
	 * Set hasHavingQual to remember if HAVING clause is present.  Needed
	 * because preprocess_expression will reduce a constant-true condition to
	 * an empty qual list ... but "HAVING TRUE" is not a semantic no-op.
	 */
	root->hasHavingQual = (parse->havingQual != NULL);

	/* Clear this flag; might get set in distribute_qual_to_rels */
	root->hasPseudoConstantQuals = false;

	/*
	 * Do expression preprocessing on targetlist and quals, as well as other
	 * random expressions in the querytree.  Note that we do not need to
	 * handle sort/group expressions explicitly, because they are actually
	 * part of the targetlist.
	 */
	parse->targetList = (List *)
		preprocess_expression(root, (Node *) parse->targetList,
							  EXPRKIND_TARGET);

	/* Constant-folding might have removed all set-returning functions */
	if (parse->hasTargetSRFs)
		parse->hasTargetSRFs = expression_returns_set((Node *) parse->targetList);

	newWithCheckOptions = NIL;
	foreach(l, parse->withCheckOptions)
	{
		WithCheckOption *wco = lfirst_node(WithCheckOption, l);

		wco->qual = preprocess_expression(root, wco->qual,
										  EXPRKIND_QUAL);
		if (wco->qual != NULL)
			newWithCheckOptions = lappend(newWithCheckOptions, wco);
	}
	parse->withCheckOptions = newWithCheckOptions;

	parse->returningList = (List *)
		preprocess_expression(root, (Node *) parse->returningList,
							  EXPRKIND_TARGET);

	preprocess_qual_conditions(root, (Node *) parse->jointree);

	parse->havingQual = preprocess_expression(root, parse->havingQual,
											  EXPRKIND_QUAL);

	foreach(l, parse->windowClause)
	{
		WindowClause *wc = lfirst_node(WindowClause, l);

		/* partitionClause/orderClause are sort/group expressions */
		wc->startOffset = preprocess_expression(root, wc->startOffset,
												EXPRKIND_LIMIT);
		wc->endOffset = preprocess_expression(root, wc->endOffset,
											  EXPRKIND_LIMIT);
	}

	parse->limitOffset = preprocess_expression(root, parse->limitOffset,
											   EXPRKIND_LIMIT);
	parse->limitCount = preprocess_expression(root, parse->limitCount,
											  EXPRKIND_LIMIT);

	if (parse->onConflict)
	{
		parse->onConflict->arbiterElems = (List *)
			preprocess_expression(root,
								  (Node *) parse->onConflict->arbiterElems,
								  EXPRKIND_ARBITER_ELEM);
		parse->onConflict->arbiterWhere =
			preprocess_expression(root,
								  parse->onConflict->arbiterWhere,
								  EXPRKIND_QUAL);
		parse->onConflict->onConflictSet = (List *)
			preprocess_expression(root,
								  (Node *) parse->onConflict->onConflictSet,
								  EXPRKIND_TARGET);
		parse->onConflict->onConflictWhere =
			preprocess_expression(root,
								  parse->onConflict->onConflictWhere,
								  EXPRKIND_QUAL);
		/* exclRelTlist contains only Vars, so no preprocessing needed */
	}

	root->append_rel_list = (List *)
		preprocess_expression(root, (Node *) root->append_rel_list,
							  EXPRKIND_APPINFO);

	/* Also need to preprocess expressions within RTEs */
	foreach(l, parse->rtable)
	{
		RangeTblEntry *rte = lfirst_node(RangeTblEntry, l);
		int			kind;
		ListCell   *lcsq;

		if (rte->rtekind == RTE_RELATION)
		{
			if (rte->tablesample)
				rte->tablesample = (TableSampleClause *)
					preprocess_expression(root,
										  (Node *) rte->tablesample,
										  EXPRKIND_TABLESAMPLE);
		}
		else if (rte->rtekind == RTE_SUBQUERY)
		{
			/*
			 * We don't want to do all preprocessing yet on the subquery's
			 * expressions, since that will happen when we plan it.  But if it
			 * contains any join aliases of our level, those have to get
			 * expanded now, because planning of the subquery won't do it.
			 * That's only possible if the subquery is LATERAL.
			 */
			if (rte->lateral && root->hasJoinRTEs)
				rte->subquery = (Query *)
					flatten_join_alias_vars(root->parse,
											(Node *) rte->subquery);
		}
		else if (rte->rtekind == RTE_FUNCTION)
		{
			/* Preprocess the function expression(s) fully */
			kind = rte->lateral ? EXPRKIND_RTFUNC_LATERAL : EXPRKIND_RTFUNC;
			rte->functions = (List *)
				preprocess_expression(root, (Node *) rte->functions, kind);
		}
		else if (rte->rtekind == RTE_TABLEFUNC)
		{
			/* Preprocess the function expression(s) fully */
			kind = rte->lateral ? EXPRKIND_TABLEFUNC_LATERAL : EXPRKIND_TABLEFUNC;
			rte->tablefunc = (TableFunc *)
				preprocess_expression(root, (Node *) rte->tablefunc, kind);
		}
		else if (rte->rtekind == RTE_VALUES)
		{
			/* Preprocess the values lists fully */
			kind = rte->lateral ? EXPRKIND_VALUES_LATERAL : EXPRKIND_VALUES;
			rte->values_lists = (List *)
				preprocess_expression(root, (Node *) rte->values_lists, kind);
		}

		/*
		 * Process each element of the securityQuals list as if it were a
		 * separate qual expression (as indeed it is).  We need to do it this
		 * way to get proper canonicalization of AND/OR structure.  Note that
		 * this converts each element into an implicit-AND sublist.
		 */
		foreach(lcsq, rte->securityQuals)
		{
			lfirst(lcsq) = preprocess_expression(root,
												 (Node *) lfirst(lcsq),
												 EXPRKIND_QUAL);
		}
	}

	/*
	 * Now that we are done preprocessing expressions, and in particular done
	 * flattening join alias variables, get rid of the joinaliasvars lists.
	 * They no longer match what expressions in the rest of the tree look
	 * like, because we have not preprocessed expressions in those lists (and
	 * do not want to; for example, expanding a SubLink there would result in
	 * a useless unreferenced subplan).  Leaving them in place simply creates
	 * a hazard for later scans of the tree.  We could try to prevent that by
	 * using QTW_IGNORE_JOINALIASES in every tree scan done after this point,
	 * but that doesn't sound very reliable.
	 */
	if (root->hasJoinRTEs)
	{
		foreach(l, parse->rtable)
		{
			RangeTblEntry *rte = lfirst_node(RangeTblEntry, l);

			rte->joinaliasvars = NIL;
		}
	}

	/*
	 * In some cases we may want to transfer a HAVING clause into WHERE. We
	 * cannot do so if the HAVING clause contains aggregates (obviously) or
	 * volatile functions (since a HAVING clause is supposed to be executed
	 * only once per group).  We also can't do this if there are any nonempty
	 * grouping sets; moving such a clause into WHERE would potentially change
	 * the results, if any referenced column isn't present in all the grouping
	 * sets.  (If there are only empty grouping sets, then the HAVING clause
	 * must be degenerate as discussed below.)
	 *
	 * Also, it may be that the clause is so expensive to execute that we're
	 * better off doing it only once per group, despite the loss of
	 * selectivity.  This is hard to estimate short of doing the entire
	 * planning process twice, so we use a heuristic: clauses containing
	 * subplans are left in HAVING.  Otherwise, we move or copy the HAVING
	 * clause into WHERE, in hopes of eliminating tuples before aggregation
	 * instead of after.
	 *
	 * If the query has explicit grouping then we can simply move such a
	 * clause into WHERE; any group that fails the clause will not be in the
	 * output because none of its tuples will reach the grouping or
	 * aggregation stage.  Otherwise we must have a degenerate (variable-free)
	 * HAVING clause, which we put in WHERE so that query_planner() can use it
	 * in a gating Result node, but also keep in HAVING to ensure that we
	 * don't emit a bogus aggregated row. (This could be done better, but it
	 * seems not worth optimizing.)
	 *
	 * Note that both havingQual and parse->jointree->quals are in
	 * implicitly-ANDed-list form at this point, even though they are declared
	 * as Node *.
	 */
	newHaving = NIL;
	foreach(l, (List *) parse->havingQual)
	{
		Node	   *havingclause = (Node *) lfirst(l);

		if ((parse->groupClause && parse->groupingSets) ||
			contain_agg_clause(havingclause) ||
			contain_volatile_functions(havingclause) ||
			contain_subplans(havingclause))
		{
			/* keep it in HAVING */
			newHaving = lappend(newHaving, havingclause);
		}
		else if (parse->groupClause && !parse->groupingSets)
		{
			/* move it to WHERE */
			parse->jointree->quals = (Node *)
				lappend((List *) parse->jointree->quals, havingclause);
		}
		else
		{
			/* put a copy in WHERE, keep it in HAVING */
			parse->jointree->quals = (Node *)
				lappend((List *) parse->jointree->quals,
						copyObject(havingclause));
			newHaving = lappend(newHaving, havingclause);
		}
	}
	parse->havingQual = (Node *) newHaving;

	/* Remove any redundant GROUP BY columns */
	remove_useless_groupby_columns(root);

	/*
	 * If we have any outer joins, try to reduce them to plain inner joins.
	 * This step is most easily done after we've done expression
	 * preprocessing.
	 */
	if (hasOuterJoins)
		reduce_outer_joins(root);

	/*
	 * If we have any RTE_RESULT relations, see if they can be deleted from
	 * the jointree.  This step is most effectively done after we've done
	 * expression preprocessing and outer join reduction.
	 */
	if (hasResultRTEs)
		remove_useless_result_rtes(root);

	/*
	 * Do the main planning.  If we have an inherited target relation, that
	 * needs special processing, else go straight to grouping_planner.
	 */
	if (parse->resultRelation &&
		rt_fetch(parse->resultRelation, parse->rtable)->inh)
		inheritance_planner(root);
	else
		grouping_planner(root, false, tuple_fraction);

	/*
	 * Capture the set of outer-level param IDs we have access to, for use in
	 * extParam/allParam calculations later.
	 */
	SS_identify_outer_params(root);

	/*
	 * If any initPlans were created in this query level, adjust the surviving
	 * Paths' costs and parallel-safety flags to account for them.  The
	 * initPlans won't actually get attached to the plan tree till
	 * create_plan() runs, but we must include their effects now.
	 */
	final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);
	SS_charge_for_initplans(root, final_rel);

	/*
	 * Make sure we've identified the cheapest Path for the final rel.  (By
	 * doing this here not in grouping_planner, we include initPlan costs in
	 * the decision, though it's unlikely that will change anything.)
	 */
	set_cheapest(final_rel);

	return root;
}
```

### pull_up_sublinks函数

```c
	/*
	 * Look for ANY and EXISTS SubLinks in WHERE and JOIN/ON clauses, and try
	 * to transform them into joins.  Note that this step does not descend
	 * into subqueries; if we pull up any subqueries below, their SubLinks are
	 * processed just before pulling them up.
	 */
	if (parse->hasSubLinks)
		pull_up_sublinks(root);

void
pull_up_sublinks(PlannerInfo *root)
{
	Node	   *jtnode;
	Relids		relids;

	/* Begin recursion through the jointree */
	jtnode = pull_up_sublinks_jointree_recurse(root,
											   (Node *) root->parse->jointree,
											   &relids);

	/*
	 * root->parse->jointree must always be a FromExpr, so insert a dummy one
	 * if we got a bare RangeTblRef or JoinExpr out of the recursion.
	 */
	if (IsA(jtnode, FromExpr))
		root->parse->jointree = (FromExpr *) jtnode;
	else
		root->parse->jointree = makeFromExpr(list_make1(jtnode), NULL);
}
//真正处理的优化操作的是pull_up_sublinks_jointree_recurse
```

## 二、子链接优化步骤

PostgreSQL会遍历WHERE和JOIN/ON子句中的ANY/EXISTS类型的子链接，如果满足条件会将这些子链接转化为SEMI-JOIN或者ANTI-SEMI-JOIN类型的子句。下面有两个问题会在结尾给出答案。

### 问题

- 为什么要将子链接转换为join的方式，这样做有什么好处？
- 子链接转换为join有哪些条件？

### pull_up_sublinks_jointree_recurse函数

```
static Node *
pull_up_sublinks_jointree_recurse(PlannerInfo *root, Node *jtnode,
								  Relids *relids)
{
	if (jtnode == NULL)
	{
		*relids = NULL;
	}
	else if (IsA(jtnode, RangeTblRef))
	{
		int			varno = ((RangeTblRef *) jtnode)->rtindex;

		*relids = bms_make_singleton(varno);
		/* jtnode is returned unmodified */
	}
	else if (IsA(jtnode, FromExpr))
	{
		FromExpr   *f = (FromExpr *) jtnode;
		List	   *newfromlist = NIL;
		Relids		frelids = NULL;
		FromExpr   *newf;
		Node	   *jtlink;
		ListCell   *l;

		/* First, recurse to process children and collect their relids */
		foreach(l, f->fromlist)
		{
			Node	   *newchild;
			Relids		childrelids;

			newchild = pull_up_sublinks_jointree_recurse(root,
														 lfirst(l),
														 &childrelids);
			newfromlist = lappend(newfromlist, newchild);
			frelids = bms_join(frelids, childrelids);
		}
		/* Build the replacement FromExpr; no quals yet */
		newf = makeFromExpr(newfromlist, NULL);
		/* Set up a link representing the rebuilt jointree */
		jtlink = (Node *) newf;
		/* Now process qual --- all children are available for use */
		newf->quals = pull_up_sublinks_qual_recurse(root, f->quals,
													&jtlink, frelids,
													NULL, NULL);

		/*
		 * Note that the result will be either newf, or a stack of JoinExprs
		 * with newf at the base.  We rely on subsequent optimization steps to
		 * flatten this and rearrange the joins as needed.
		 *
		 * Although we could include the pulled-up subqueries in the returned
		 * relids, there's no need since upper quals couldn't refer to their
		 * outputs anyway.
		 */
		*relids = frelids;
		jtnode = jtlink;
	}
	else if (IsA(jtnode, JoinExpr))
	{
		JoinExpr   *j;
		Relids		leftrelids;
		Relids		rightrelids;
		Node	   *jtlink;

		/*
		 * Make a modifiable copy of join node, but don't bother copying its
		 * subnodes (yet).
		 */
		j = (JoinExpr *) palloc(sizeof(JoinExpr));
		memcpy(j, jtnode, sizeof(JoinExpr));
		jtlink = (Node *) j;

		/* Recurse to process children and collect their relids */
		j->larg = pull_up_sublinks_jointree_recurse(root, j->larg,
													&leftrelids);
		j->rarg = pull_up_sublinks_jointree_recurse(root, j->rarg,
													&rightrelids);

		/*
		 * Now process qual, showing appropriate child relids as available,
		 * and attach any pulled-up jointree items at the right place. In the
		 * inner-join case we put new JoinExprs above the existing one (much
		 * as for a FromExpr-style join).  In outer-join cases the new
		 * JoinExprs must go into the nullable side of the outer join. The
		 * point of the available_rels machinations is to ensure that we only
		 * pull up quals for which that's okay.
		 *
		 * We don't expect to see any pre-existing JOIN_SEMI or JOIN_ANTI
		 * nodes here.
		 */
		switch (j->jointype)
		{
			case JOIN_INNER:
				j->quals = pull_up_sublinks_qual_recurse(root, j->quals,
														 &jtlink,
														 bms_union(leftrelids,
																   rightrelids),
														 NULL, NULL);
				break;
			case JOIN_LEFT:
				j->quals = pull_up_sublinks_qual_recurse(root, j->quals,
														 &j->rarg,
														 rightrelids,
														 NULL, NULL);
				break;
			case JOIN_FULL:
				/* can't do anything with full-join quals */
				break;
			case JOIN_RIGHT:
				j->quals = pull_up_sublinks_qual_recurse(root, j->quals,
														 &j->larg,
														 leftrelids,
														 NULL, NULL);
				break;
			default:
				elog(ERROR, "unrecognized join type: %d",
					 (int) j->jointype);
				break;
		}

		/*
		 * Although we could include the pulled-up subqueries in the returned
		 * relids, there's no need since upper quals couldn't refer to their
		 * outputs anyway.  But we *do* need to include the join's own rtindex
		 * because we haven't yet collapsed join alias variables, so upper
		 * levels would mistakenly think they couldn't use references to this
		 * join.
		 */
		*relids = bms_join(leftrelids, rightrelids);
		if (j->rtindex)
			*relids = bms_add_member(*relids, j->rtindex);
		jtnode = jtlink;
	}
	else
		elog(ERROR, "unrecognized node type: %d",
			 (int) nodeTag(jtnode));
	return jtnode;
}
```

根据上面代码分析可知pull_up_sublinks_jointree_recurse函数根据节点的类型来分类处理：

- RangeTblRef:不做优化。
- FromExpr:如果是fromlist，会遍历fromlist递归调用pull_up_sublinks_jointree-recurse函数来处理；如果是quals,则调用pull_up_sublinks_qual_recurse来处理。
- JoinExpr：调用pull_up_sublinks_jointree_recurse函数处理左右子树，调用pull_up_sublinks_qual_recurse来处理quals。

### pull_up_sublinks_qual_recurse函数

```c
static Node *
pull_up_sublinks_qual_recurse(PlannerInfo *root, Node *node,
							  Node **jtlink1, Relids available_rels1,
							  Node **jtlink2, Relids available_rels2)
{
	if (node == NULL)
		return NULL;
	if (IsA(node, SubLink))
	{
		SubLink    *sublink = (SubLink *) node;
		JoinExpr   *j;
		Relids		child_rels;

		/* Is it a convertible ANY or EXISTS clause? */
		if (sublink->subLinkType == ANY_SUBLINK)
		{
			if ((j = convert_ANY_sublink_to_join(root, sublink,
												 available_rels1)) != NULL)
			{
				/* Yes; insert the new join node into the join tree */
				j->larg = *jtlink1;
				*jtlink1 = (Node *) j;
				/* Recursively process pulled-up jointree nodes */
				j->rarg = pull_up_sublinks_jointree_recurse(root,
															j->rarg,
															&child_rels);

				/*
				 * Now recursively process the pulled-up quals.  Any inserted
				 * joins can get stacked onto either j->larg or j->rarg,
				 * depending on which rels they reference.
				 */
				j->quals = pull_up_sublinks_qual_recurse(root,
														 j->quals,
														 &j->larg,
														 available_rels1,
														 &j->rarg,
														 child_rels);
				/* Return NULL representing constant TRUE */
				return NULL;
			}
			if (available_rels2 != NULL &&
				(j = convert_ANY_sublink_to_join(root, sublink,
												 available_rels2)) != NULL)
			{
				/* Yes; insert the new join node into the join tree */
				j->larg = *jtlink2;
				*jtlink2 = (Node *) j;
				/* Recursively process pulled-up jointree nodes */
				j->rarg = pull_up_sublinks_jointree_recurse(root,
															j->rarg,
															&child_rels);

				/*
				 * Now recursively process the pulled-up quals.  Any inserted
				 * joins can get stacked onto either j->larg or j->rarg,
				 * depending on which rels they reference.
				 */
				j->quals = pull_up_sublinks_qual_recurse(root,
														 j->quals,
														 &j->larg,
														 available_rels2,
														 &j->rarg,
														 child_rels);
				/* Return NULL representing constant TRUE */
				return NULL;
			}
		}
		else if (sublink->subLinkType == EXISTS_SUBLINK)
		{
			if ((j = convert_EXISTS_sublink_to_join(root, sublink, false,
													available_rels1)) != NULL)
			{
				/* Yes; insert the new join node into the join tree */
				j->larg = *jtlink1;
				*jtlink1 = (Node *) j;
				/* Recursively process pulled-up jointree nodes */
				j->rarg = pull_up_sublinks_jointree_recurse(root,
															j->rarg,
															&child_rels);

				/*
				 * Now recursively process the pulled-up quals.  Any inserted
				 * joins can get stacked onto either j->larg or j->rarg,
				 * depending on which rels they reference.
				 */
				j->quals = pull_up_sublinks_qual_recurse(root,
														 j->quals,
														 &j->larg,
														 available_rels1,
														 &j->rarg,
														 child_rels);
				/* Return NULL representing constant TRUE */
				return NULL;
			}
			if (available_rels2 != NULL &&
				(j = convert_EXISTS_sublink_to_join(root, sublink, false,
													available_rels2)) != NULL)
			{
				/* Yes; insert the new join node into the join tree */
				j->larg = *jtlink2;
				*jtlink2 = (Node *) j;
				/* Recursively process pulled-up jointree nodes */
				j->rarg = pull_up_sublinks_jointree_recurse(root,
															j->rarg,
															&child_rels);

				/*
				 * Now recursively process the pulled-up quals.  Any inserted
				 * joins can get stacked onto either j->larg or j->rarg,
				 * depending on which rels they reference.
				 */
				j->quals = pull_up_sublinks_qual_recurse(root,
														 j->quals,
														 &j->larg,
														 available_rels2,
														 &j->rarg,
														 child_rels);
				/* Return NULL representing constant TRUE */
				return NULL;
			}
		}
		/* Else return it unmodified */
		return node;
	}
	if (is_notclause(node))
	{
		/* If the immediate argument of NOT is EXISTS, try to convert */
		SubLink    *sublink = (SubLink *) get_notclausearg((Expr *) node);
		JoinExpr   *j;
		Relids		child_rels;

		if (sublink && IsA(sublink, SubLink))
		{
			if (sublink->subLinkType == EXISTS_SUBLINK)
			{
				if ((j = convert_EXISTS_sublink_to_join(root, sublink, true,
														available_rels1)) != NULL)
				{
					/* Yes; insert the new join node into the join tree */
					j->larg = *jtlink1;
					*jtlink1 = (Node *) j;
					/* Recursively process pulled-up jointree nodes */
					j->rarg = pull_up_sublinks_jointree_recurse(root,
																j->rarg,
																&child_rels);

					/*
					 * Now recursively process the pulled-up quals.  Because
					 * we are underneath a NOT, we can't pull up sublinks that
					 * reference the left-hand stuff, but it's still okay to
					 * pull up sublinks referencing j->rarg.
					 */
					j->quals = pull_up_sublinks_qual_recurse(root,
															 j->quals,
															 &j->rarg,
															 child_rels,
															 NULL, NULL);
					/* Return NULL representing constant TRUE */
					return NULL;
				}
				if (available_rels2 != NULL &&
					(j = convert_EXISTS_sublink_to_join(root, sublink, true,
														available_rels2)) != NULL)
				{
					/* Yes; insert the new join node into the join tree */
					j->larg = *jtlink2;
					*jtlink2 = (Node *) j;
					/* Recursively process pulled-up jointree nodes */
					j->rarg = pull_up_sublinks_jointree_recurse(root,
																j->rarg,
																&child_rels);

					/*
					 * Now recursively process the pulled-up quals.  Because
					 * we are underneath a NOT, we can't pull up sublinks that
					 * reference the left-hand stuff, but it's still okay to
					 * pull up sublinks referencing j->rarg.
					 */
					j->quals = pull_up_sublinks_qual_recurse(root,
															 j->quals,
															 &j->rarg,
															 child_rels,
															 NULL, NULL);
					/* Return NULL representing constant TRUE */
					return NULL;
				}
			}
		}
		/* Else return it unmodified */
		return node;
	}
	if (is_andclause(node))
	{
		/* Recurse into AND clause */
		List	   *newclauses = NIL;
		ListCell   *l;

		foreach(l, ((BoolExpr *) node)->args)
		{
			Node	   *oldclause = (Node *) lfirst(l);
			Node	   *newclause;

			newclause = pull_up_sublinks_qual_recurse(root,
													  oldclause,
													  jtlink1,
													  available_rels1,
													  jtlink2,
													  available_rels2);
			if (newclause)
				newclauses = lappend(newclauses, newclause);
		}
		/* We might have got back fewer clauses than we started with */
		if (newclauses == NIL)
			return NULL;
		else if (list_length(newclauses) == 1)
			return (Node *) linitial(newclauses);
		else
			return (Node *) make_andclause(newclauses);
	}
	/* Stop if not an AND */
	return node;
}
```

pull_up_sublinks_qual_recurse函数会处理一下类型的语句:

- AND语句:调用pull_up_sublinks_qual_recurse函数遍历的处理AND节点每一项。
- NOT语句:如果有NOT EXISTS，需要调用函数get_notclausearg进行转换，然后再根据子链接的类型进行分类进行转换。
- sublink语句:如果是EXISTS，调用convert_EXISTS_sublink_to_join函数处理；如果是ANY，调用convert_ANY_sublink_to_join函数处理。

### convert_ANY_sublink_to_join函数

```c
JoinExpr *
convert_ANY_sublink_to_join(PlannerInfo *root, SubLink *sublink,
							Relids available_rels)
{
	JoinExpr   *result;
	Query	   *parse = root->parse;
	Query	   *subselect = (Query *) sublink->subselect;
	Relids		upper_varnos;
	int			rtindex;
	RangeTblEntry *rte;
	RangeTblRef *rtr;
	List	   *subquery_vars;
	Node	   *quals;
	ParseState *pstate;

	Assert(sublink->subLinkType == ANY_SUBLINK);

	/*
	 * The sub-select must not refer to any Vars of the parent query. (Vars of
	 * higher levels should be okay, though.)
	 */
	if (contain_vars_of_level((Node *) subselect, 1))
		return NULL;

	/*
	 * The test expression must contain some Vars of the parent query, else
	 * it's not gonna be a join.  (Note that it won't have Vars referring to
	 * the subquery, rather Params.)
	 */
	upper_varnos = pull_varnos(sublink->testexpr);
	if (bms_is_empty(upper_varnos))
		return NULL;

	/*
	 * However, it can't refer to anything outside available_rels.
	 */
	if (!bms_is_subset(upper_varnos, available_rels))
		return NULL;

	/*
	 * The combining operators and left-hand expressions mustn't be volatile.
	 */
	if (contain_volatile_functions(sublink->testexpr))
		return NULL;

	/* Create a dummy ParseState for addRangeTableEntryForSubquery */
	pstate = make_parsestate(NULL);

	/*
	 * Okay, pull up the sub-select into upper range table.
	 *
	 * We rely here on the assumption that the outer query has no references
	 * to the inner (necessarily true, other than the Vars that we build
	 * below). Therefore this is a lot easier than what pull_up_subqueries has
	 * to go through.
	 */
	rte = addRangeTableEntryForSubquery(pstate,
										subselect,
										makeAlias("ANY_subquery", NIL),
										false,
										false);
	parse->rtable = lappend(parse->rtable, rte);
	rtindex = list_length(parse->rtable);

	/*
	 * Form a RangeTblRef for the pulled-up sub-select.
	 */
	rtr = makeNode(RangeTblRef);
	rtr->rtindex = rtindex;

	/*
	 * Build a list of Vars representing the subselect outputs.
	 */
	subquery_vars = generate_subquery_vars(root,
										   subselect->targetList,
										   rtindex);

	/*
	 * Build the new join's qual expression, replacing Params with these Vars.
	 */
	quals = convert_testexpr(root, sublink->testexpr, subquery_vars);

	/*
	 * And finally, build the JoinExpr node.
	 */
	result = makeNode(JoinExpr);
	result->jointype = JOIN_SEMI;
	result->isNatural = false;
	result->larg = NULL;		/* caller must fill this in */
	result->rarg = (Node *) rtr;
	result->usingClause = NIL;
	result->quals = quals;
	result->alias = NULL;
	result->rtindex = 0;		/* we don't need an RTE for it */

	return result;
}
```

### convert_ANY_sublink_to_join函数首先说明了能够将子链接转换为join必须满足的条件

1. 子链接中的子查询不包含父查询中的任何VAR变量。
2. 比较表达式必须包含父查询中一些Var变量。
3. 比较表达式不包含任何的易失函数。

### convert_ANY_sublink_to_join函数执行流程

1. 查看是否满足转换的条件
2. 将子链接中的子查询转换名为‘ANY_subquery’的RTE，并将该RTE加入到父查询的rtable中。
3. 定义一个RangeTblRef节点(rtr)，并初始化rtindex(指向父查询中的rtable)。
4. 将子查询的目标列转换成Var变量保存到subquery_vars,并将比较表达式和subquery_vars，一起形成quals。
5. 定义一个JoinExpr节点，类型为JOIN_SEMI，rlags为rtr,quals为第四步形成的新的qual表达式。

### convert_EXISTS_sublink_to_join函数

```c
JoinExpr *
convert_EXISTS_sublink_to_join(PlannerInfo *root, SubLink *sublink,
							   bool under_not, Relids available_rels)
{
	JoinExpr   *result;
	Query	   *parse = root->parse;
	Query	   *subselect = (Query *) sublink->subselect;
	Node	   *whereClause;
	int			rtoffset;
	int			varno;
	Relids		clause_varnos;
	Relids		upper_varnos;

	Assert(sublink->subLinkType == EXISTS_SUBLINK);

	/*
	 * Can't flatten if it contains WITH.  (We could arrange to pull up the
	 * WITH into the parent query's cteList, but that risks changing the
	 * semantics, since a WITH ought to be executed once per associated query
	 * call.)  Note that convert_ANY_sublink_to_join doesn't have to reject
	 * this case, since it just produces a subquery RTE that doesn't have to
	 * get flattened into the parent query.
	 */
	if (subselect->cteList)
		return NULL;

	/*
	 * Copy the subquery so we can modify it safely (see comments in
	 * make_subplan).
	 */
	subselect = copyObject(subselect);

	/*
	 * See if the subquery can be simplified based on the knowledge that it's
	 * being used in EXISTS().  If we aren't able to get rid of its
	 * targetlist, we have to fail, because the pullup operation leaves us
	 * with noplace to evaluate the targetlist.
	 */
	if (!simplify_EXISTS_query(root, subselect))
		return NULL;

	/*
	 * Separate out the WHERE clause.  (We could theoretically also remove
	 * top-level plain JOIN/ON clauses, but it's probably not worth the
	 * trouble.)
	 */
	whereClause = subselect->jointree->quals;
	subselect->jointree->quals = NULL;

	/*
	 * The rest of the sub-select must not refer to any Vars of the parent
	 * query.  (Vars of higher levels should be okay, though.)
	 */
	if (contain_vars_of_level((Node *) subselect, 1))
		return NULL;

	/*
	 * On the other hand, the WHERE clause must contain some Vars of the
	 * parent query, else it's not gonna be a join.
	 */
	if (!contain_vars_of_level(whereClause, 1))
		return NULL;

	/*
	 * We don't risk optimizing if the WHERE clause is volatile, either.
	 */
	if (contain_volatile_functions(whereClause))
		return NULL;

	/*
	 * The subquery must have a nonempty jointree, but we can make it so.
	 */
	replace_empty_jointree(subselect);

	/*
	 * Prepare to pull up the sub-select into top range table.
	 *
	 * We rely here on the assumption that the outer query has no references
	 * to the inner (necessarily true). Therefore this is a lot easier than
	 * what pull_up_subqueries has to go through.
	 *
	 * In fact, it's even easier than what convert_ANY_sublink_to_join has to
	 * do.  The machinations of simplify_EXISTS_query ensured that there is
	 * nothing interesting in the subquery except an rtable and jointree, and
	 * even the jointree FromExpr no longer has quals.  So we can just append
	 * the rtable to our own and use the FromExpr in our jointree. But first,
	 * adjust all level-zero varnos in the subquery to account for the rtable
	 * merger.
	 */
	rtoffset = list_length(parse->rtable);
	OffsetVarNodes((Node *) subselect, rtoffset, 0);
	OffsetVarNodes(whereClause, rtoffset, 0);

	/*
	 * Upper-level vars in subquery will now be one level closer to their
	 * parent than before; in particular, anything that had been level 1
	 * becomes level zero.
	 */
	IncrementVarSublevelsUp((Node *) subselect, -1, 1);
	IncrementVarSublevelsUp(whereClause, -1, 1);

	/*
	 * Now that the WHERE clause is adjusted to match the parent query
	 * environment, we can easily identify all the level-zero rels it uses.
	 * The ones <= rtoffset belong to the upper query; the ones > rtoffset do
	 * not.
	 */
	clause_varnos = pull_varnos(whereClause);
	upper_varnos = NULL;
	while ((varno = bms_first_member(clause_varnos)) >= 0)
	{
		if (varno <= rtoffset)
			upper_varnos = bms_add_member(upper_varnos, varno);
	}
	bms_free(clause_varnos);
	Assert(!bms_is_empty(upper_varnos));

	/*
	 * Now that we've got the set of upper-level varnos, we can make the last
	 * check: only available_rels can be referenced.
	 */
	if (!bms_is_subset(upper_varnos, available_rels))
		return NULL;

	/* Now we can attach the modified subquery rtable to the parent */
	parse->rtable = list_concat(parse->rtable, subselect->rtable);

	/*
	 * And finally, build the JoinExpr node.
	 */
	result = makeNode(JoinExpr);
	result->jointype = under_not ? JOIN_ANTI : JOIN_SEMI;
	result->isNatural = false;
	result->larg = NULL;		/* caller must fill this in */
	/* flatten out the FromExpr node if it's useless */
	if (list_length(subselect->jointree->fromlist) == 1)
		result->rarg = (Node *) linitial(subselect->jointree->fromlist);
	else
		result->rarg = (Node *) subselect->jointree;
	result->usingClause = NIL;
	result->quals = whereClause;
	result->alias = NULL;
	result->rtindex = 0;		/* we don't need an RTE for it */

	return result;
}
```

### convert_EXISTS_sublink_to_join函数优化的条件

1. 子链接中的子查询不包含CTE
2. 子链接中的子查询不包含父查询中的任何VAR变量。
3. WHERE子句中必须包含父查询中一些Var变量。
4. WHERE子句中不包含任何的易失函数。

### convert_EXISTS_sublink_to_join函数流程

1. 查看是否满足转换的条件
2. 提升subselect、where子句的层次(ANY没有提升子查询的层次)
3. 将子查询提升到父查询级别的范围表(EXISTS与ANY子链接不同，EXISTS可以直接展平子查询的rtable到父查询级别，而ANY只是产生RTE，并没有展平父查询)
4. 创建joinexpr节点，指定join的类型，rlags为子查询的jointree(如果单表就是fromlist，也就是父表和该表join,如果是多表，先子查询多表join然后再与父表join),quals为子查询的quals。

## 三、子链接优化实例

### 一、不优化WHERE和JOIN/ON之外的子链接

```sql
-- 示例SQL
test=# select id in (select id from st where id<4) as new_id,info from test;

-- breakpoint位置 
	if (parse->hasSubLinks)
		pull_up_sublinks(root);

-- stack
Thread #1 [postgres] 4774 [core: 3] (Suspended : Step)	
	subquery_planner() at planner.c:657 0x7d916c	
	standard_planner() at planner.c:406 0x7d87a9	
	planner() at planner.c:275 0x7d855d	
	pg_plan_query() at postgres.c:878 0x8e0631	
	pg_plan_queries() at postgres.c:968 0x8e075e	
	exec_simple_query() at postgres.c:1,143 0x8e0a2f	
	PostgresMain() at postgres.c:4,236 0x8e4e0e	
	BackendRun() at postmaster.c:4,431 0x83bb0c	
	BackendStartup() at postmaster.c:4,122 0x83b2ea	
	ServerLoop() at postmaster.c:1,704 0x8376e1	
	<...more frames...>	
	
-- 
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-115.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
Warning: the current language does not match this frame.

print parse->hasSubLinks
$5 = false

-- 说明没有处于FROM子句的子链接不被优化

-- 查看该SQL的执行计划
test=# explain analyze select id in (select id from st where id<4) as new_id,info from test;
                                             QUERY PLAN                                             
----------------------------------------------------------------------------------------------------
 Seq Scan on test  (cost=1.07..2.19 rows=10 width=5) (actual time=0.024..0.029 rows=10 loops=1)
   SubPlan 1
     ->  Seq Scan on st  (cost=0.00..1.06 rows=3 width=4) (actual time=0.004..0.006 rows=3 loops=1)
           Filter: (id < 4)
           Rows Removed by Filter: 2
 Planning Time: 0.315 ms
 Execution Time: 0.066 ms
(7 rows)
-- 可以看出该子链接没有被优化，而是以子查询的方式执行
```

### 二、ANY类型的子链接的处理

```sql
-- 示例SQL
test=# select test.id,test.info,sub.item from test,(select * from st where id in (select id from tmp where id<3)) as sub where test.id in (select id from tmp where temp=sub.item);


-- stack
Thread #1 [postgres] 5197 [core: 0] (Suspended : Step)	
	pull_up_sublinks() at prepjointree.c:196 0x7ed3e6	
	subquery_planner() at planner.c:658 0x7d9187	
	standard_planner() at planner.c:406 0x7d87a9	
	planner() at planner.c:275 0x7d855d	
	pg_plan_query() at postgres.c:878 0x8e0631	
	pg_plan_queries() at postgres.c:968 0x8e075e	
	exec_simple_query() at postgres.c:1,143 0x8e0a2f	
	PostgresMain() at postgres.c:4,236 0x8e4e0e	
	BackendRun() at postmaster.c:4,431 0x83bb0c	
	BackendStartup() at postmaster.c:4,122 0x83b2ea	
	<...more frames...>	

-- 分析
首先进入第一层优化:对于test.id in (select id from tmp where temp=sub.item);子链接
	/*
	 * The sub-select must not refer to any Vars of the parent query. (Vars of
	 * higher levels should be okay, though.)
	 */
	if (contain_vars_of_level((Node *) subselect, 1))
		return NULL;
不满足优化的条件所以该子链接不能被优化

最后进入第二层查询：(select * from st where id in (select id from tmp where id<3)) as sub
该子查询包含子链接，但是再处理外层子链接时，该子查询为RangeTblRef类型。进入第二层优化是使用pull_up_subqueries函数，对子查询进行优化(实际优化函数为pull_up_subqueries_recurse)。

static Node *
pull_up_subqueries_recurse(PlannerInfo *root, Node *jtnode,
						   JoinExpr *lowest_outer_join,
						   JoinExpr *lowest_nulling_outer_join,
						   AppendRelInfo *containing_appendrel)
{
	Assert(jtnode != NULL);
	if (IsA(jtnode, RangeTblRef))
	{
		int			varno = ((RangeTblRef *) jtnode)->rtindex;
		RangeTblEntry *rte = rt_fetch(varno, root->parse->rtable);

		/*
		 * Is this a subquery RTE, and if so, is the subquery simple enough to
		 * pull up?
		 *
		 * If we are looking at an append-relation member, we can't pull it up
		 * unless is_safe_append_member says so.
		 */
		if (rte->rtekind == RTE_SUBQUERY &&
			is_simple_subquery(rte->subquery, rte, lowest_outer_join) &&
			(containing_appendrel == NULL ||
			 is_safe_append_member(rte->subquery)))
			return pull_up_simple_subquery(root, jtnode, rte,
										   lowest_outer_join,
										   lowest_nulling_outer_join,
                	                        containing_appendrel);
		......
}
-- 可以看到最后定位到pull_up_simple_subquery函数
	/*
	 * Pull up any SubLinks within the subquery's quals, so that we don't
	 * leave unoptimized SubLinks behind.
	 */
	if (subquery->hasSubLinks)
		pull_up_sublinks(subroot);
-- 最后对于该子查询中的子链接函数又调用了pull_up_sublinks函数来对其优化，优化的过程与父查询一致。



-- 查看该SQL的执行计划
test=# explain analyze select test.id,test.info,sub.item from test,(select * from st where id in (select id from tmp where id<3)) as sub where test.id in (select id from tmp where temp=sub.item);
                                                  QUERY PLAN                                                   
---------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=1.12..9.10 rows=3 width=18) (actual time=0.106..0.106 rows=0 loops=1)
   Join Filter: (SubPlan 1)
   Rows Removed by Join Filter: 20
   ->  Hash Semi Join  (cost=1.12..2.20 rows=1 width=10) (actual time=0.030..0.033 rows=2 loops=1)
         Hash Cond: (st.id = tmp.id)
         ->  Seq Scan on st  (cost=0.00..1.05 rows=5 width=14) (actual time=0.011..0.012 rows=5 loops=1)
         ->  Hash  (cost=1.11..1.11 rows=1 width=4) (actual time=0.012..0.012 rows=2 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 9kB
               ->  Seq Scan on tmp  (cost=0.00..1.11 rows=1 width=4) (actual time=0.006..0.009 rows=2 loops=1)
                     Filter: (id < 3)
                     Rows Removed by Filter: 7
   ->  Seq Scan on test  (cost=0.00..1.10 rows=10 width=8) (actual time=0.002..0.003 rows=10 loops=2)
   SubPlan 1
     ->  Seq Scan on tmp tmp_1  (cost=0.00..1.11 rows=9 width=4) (actual time=0.002..0.002 rows=0 loops=20)
           Filter: (temp = (st.item)::text)
           Rows Removed by Filter: 9
 Planning Time: 0.509 ms
 Execution Time: 0.153 ms
(18 rows)

-- 通过执行计划可以看出test.id in (select id from tmp where temp=sub.item)该子链接没有被优化，而位于子查询中的子链接优化为semi-join,可以看到优化为join的方式后，可以进一步的使用hash join。而这也是为什么需要将子链接优化成join的方式的原因

-- 注意in等价于any，所以可以优化；not in等价于！all(all类型的子链接不被优化)
```

### 三、EXISTS类型的子链接的处理

```sql
-- 示例SQL
select info from test where exists(select 1 from st where st.id=test.id);

-- stack
Thread #1 [postgres] 5197 [core: 0] (Suspended : Step)	
	convert_EXISTS_sublink_to_join() at subselect.c:1,321 0x7ea9e9	
	pull_up_sublinks_qual_recurse() at prepjointree.c:445 0x7ed9e7	
	pull_up_sublinks_jointree_recurse() at prepjointree.c:262 0x7ed5ad	
	pull_up_sublinks() at prepjointree.c:201 0x7ed412	
	subquery_planner() at planner.c:658 0x7d9187	
	standard_planner() at planner.c:406 0x7d87a9	
	planner() at planner.c:275 0x7d855d	
	pg_plan_query() at postgres.c:878 0x8e0631	
	pg_plan_queries() at postgres.c:968 0x8e075e	
	exec_simple_query() at postgres.c:1,143 0x8e0a2f	
	<...more frames...>	
	

-- 执行计划
test=# explain analyze select info from test where exists(select 1 from st where st.id=test.id);
                                               QUERY PLAN                                               
--------------------------------------------------------------------------------------------------------
 Hash Semi Join  (cost=1.11..2.29 rows=5 width=4) (actual time=0.049..0.053 rows=5 loops=1)
   Hash Cond: (test.id = st.id)
   ->  Seq Scan on test  (cost=0.00..1.10 rows=10 width=8) (actual time=0.021..0.022 rows=10 loops=1)
   ->  Hash  (cost=1.05..1.05 rows=5 width=4) (actual time=0.015..0.015 rows=5 loops=1)
         Buckets: 1024  Batches: 1  Memory Usage: 9kB
         ->  Seq Scan on st  (cost=0.00..1.05 rows=5 width=4) (actual time=0.007..0.008 rows=5 loops=1)
 Planning Time: 0.514 ms
 Execution Time: 0.104 ms
(8 rows)
```

## 四、问题回答

### 一、为什么要将子链接转换为join的方式，这样做有什么好处？

1. 首先semi-join定义满足子链接的要求
2. 将子链接转化为join的方式，为了后续Hash join、Merge join优化提供了可能性。如果没有优化成join的方式，就相当于是执行子查询始终是NestLoop join。

### 二、满足什么条件可以优化为join?

上面已经给出了可以优化的条件。

## 五、子链接转化为join的步骤

上面已经给出了ANY/EXISTS的优化步骤(joinexpr的quals,rlags已经指定)，下面给出后续的步骤。

```c
/* Yes; insert the new join node into the join tree */
				j->larg = *jtlink1;
				*jtlink1 = (Node *) j;
				/* Recursively process pulled-up jointree nodes */
				j->rarg = pull_up_sublinks_jointree_recurse(root,
															j->rarg,
															&child_rels);

				/*
				 * Now recursively process the pulled-up quals.  Any inserted
				 * joins can get stacked onto either j->larg or j->rarg,
				 * depending on which rels they reference.
				 */
				j->quals = pull_up_sublinks_qual_recurse(root,
														 j->quals,
														 &j->larg,
														 available_rels1,
														 &j->rarg,
														 child_rels);
				/* Return NULL representing constant TRUE */
				return NULL;
/*
可以看到joinexpr节点的larg指定为jointree的fromlist(已经经过pull_up_sublink优化,如果from子查询中包含子链接会在后面继续优化的)，并且调用函数pull_up_sublinks_jointree_recurse继续对rarg和quals优化的
*/
```

