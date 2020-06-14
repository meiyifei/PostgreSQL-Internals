# 	PostgreSQL子查询优化

## 一、入口函数

遍历jointree进行子链接优化，此时jointree满足要求的子链接都已经上提，但是有个问题：joinexpr的larg中fromlist中包含的子查询以及rarg中存在的Rangetblref(也是一个子查询)该怎样来处理呢？

对于这些出现在jointree中的子查询,pg_pull_up_subqueries会检查这个子查询所涉及到的范围表是否能够合并到父查询中(消除子查询)，如果优化满足条件，则将这些子查询涉及到的范围表合并到父查询中。

```c
PlannerInfo *
subquery_planner(PlannerGlobal *glob, Query *parse,
				 PlannerInfo *parent_root,
				 bool hasRecursion, double tuple_fraction)
{
    ''''''
    /*
	 * Check to see if any subqueries in the jointree can be merged into this
	 * query.
	*/
	//遍历jointree 
	pull_up_subqueries(root);
    
    ''''''    
}

/*
 * pull_up_subqueries
 *		Look for subqueries in the rangetable that can be pulled up into
 *		the parent query.  If the subquery has no special features like
 *		grouping/aggregation then we can merge it into the parent's jointree.
 *		Also, subqueries that are simple UNION ALL structures can be
 *		converted into "append relations".
 */
void
pull_up_subqueries(PlannerInfo *root)
{
	/* Top level of jointree must always be a FromExpr */
	Assert(IsA(root->parse->jointree, FromExpr));
	/* Recursion starts with no containing join nor appendrel */
	root->parse->jointree = (FromExpr *)
		pull_up_subqueries_recurse(root, (Node *) root->parse->jointree,
								   NULL, NULL, NULL);
	/* We should still have a FromExpr */
	Assert(IsA(root->parse->jointree, FromExpr));
}



//实际的调用pull_up_subqueries_recurse函数来处理子查询的优化
```

## 二、pull_up_subqueries_recurse函数

```sql
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

		/*
		 * Alternatively, is it a simple UNION ALL subquery?  If so, flatten
		 * into an "append relation".
		 *
		 * It's safe to do this regardless of whether this query is itself an
		 * appendrel member.  (If you're thinking we should try to flatten the
		 * two levels of appendrel together, you're right; but we handle that
		 * in set_append_rel_pathlist, not here.)
		 */
		if (rte->rtekind == RTE_SUBQUERY &&
			is_simple_union_all(rte->subquery))
			return pull_up_simple_union_all(root, jtnode, rte);

		/*
		 * Or perhaps it's a simple VALUES RTE?
		 *
		 * We don't allow VALUES pullup below an outer join nor into an
		 * appendrel (such cases are impossible anyway at the moment).
		 */
		if (rte->rtekind == RTE_VALUES &&
			lowest_outer_join == NULL &&
			containing_appendrel == NULL &&
			is_simple_values(root, rte))
			return pull_up_simple_values(root, jtnode, rte);

		/* Otherwise, do nothing at this node. */
	}
	else if (IsA(jtnode, FromExpr))
	{
		FromExpr   *f = (FromExpr *) jtnode;
		ListCell   *l;

		Assert(containing_appendrel == NULL);
		/* Recursively transform all the child nodes */
		foreach(l, f->fromlist)
		{
			lfirst(l) = pull_up_subqueries_recurse(root, lfirst(l),
												   lowest_outer_join,
												   lowest_nulling_outer_join,
												   NULL);
		}
	}
	else if (IsA(jtnode, JoinExpr))
	{
		JoinExpr   *j = (JoinExpr *) jtnode;

		Assert(containing_appendrel == NULL);
		/* Recurse, being careful to tell myself when inside outer join */
		switch (j->jointype)
		{
			case JOIN_INNER:
				j->larg = pull_up_subqueries_recurse(root, j->larg,
													 lowest_outer_join,
													 lowest_nulling_outer_join,
													 NULL);
				j->rarg = pull_up_subqueries_recurse(root, j->rarg,
													 lowest_outer_join,
													 lowest_nulling_outer_join,
													 NULL);
				break;
			case JOIN_LEFT:
			case JOIN_SEMI:
			case JOIN_ANTI:
				j->larg = pull_up_subqueries_recurse(root, j->larg,
													 j,
													 lowest_nulling_outer_join,
													 NULL);
				j->rarg = pull_up_subqueries_recurse(root, j->rarg,
													 j,
													 j,
													 NULL);
				break;
			case JOIN_FULL:
				j->larg = pull_up_subqueries_recurse(root, j->larg,
													 j,
													 j,
													 NULL);
				j->rarg = pull_up_subqueries_recurse(root, j->rarg,
													 j,
													 j,
													 NULL);
				break;
			case JOIN_RIGHT:
				j->larg = pull_up_subqueries_recurse(root, j->larg,
													 j,
													 j,
													 NULL);
				j->rarg = pull_up_subqueries_recurse(root, j->rarg,
													 j,
													 lowest_nulling_outer_join,
													 NULL);
				break;
			default:
				elog(ERROR, "unrecognized join type: %d",
					 (int) j->jointype);
				break;
		}
	}
	else
		elog(ERROR, "unrecognized node type: %d",
			 (int) nodeTag(jtnode));
	return jtnode;
}
```

与子链接优化处理一样，对于遍历jointree来说需要对jointree的子节点分类型来处理。

### FromExpr节点的处理

递归的调用pull_up_subqueries_recurse函数处理FromExpr的fromlist子节点。

### JoinExpr节点的处理

子链接优化后FromExpr的fromlist为JoinExpr类型，根据JoinExpr的jointype分类的处理。递归的调用pull_up_subqueries_recurse函数处理左右子树节点。

```c
/*

lowest_outer_join、lowest_nulling_outer_join参数的含义 

If this jointree node is within either side of an outer join, then
lowest_outer_join references the lowest such JoinExpr node; otherwise
it is NULL.  We use this to constrain the effects of LATERAL subqueries.

If this jointree node is within the nullable side of an outer join, then
lowest_nulling_outer_join references the lowest such JoinExpr node;
otherwise it is NULL.  This forces use of the PlaceHolderVar mechanism for
references to non-nullable targetlist items, but only for references above
that join.

*/
```

### RangeTblRef节点的处理

1. 根据rtindex从rtable获取RangeTblRef对于的RangeTableEntry

2. 如果RTE是子查询，且该子查询满足is_simple_subquer定义的优化条件等，则调用函数pull_up_simple_subquery来处理；

   如果RTE是子查询，但是该子查询包含union all,如果满足is_simple_union_all函数定义的优化条件，则调用函数pull_up_simple_union_all来处理；

   如果RTE是值表达式，并且满足is_simple_values函数定义的优化条件等，则调用函数pull_up_simple_values来处理。

## 三、子查询优化实例

### 一、RTE为子查询

#### 优化需要满足的条件：

- 子查询为SELECT类型
- 子查询不含集合操作(指union all之外的集合操作，如果是union all有其他函数来优化)
- 子查询不能有聚集操作、窗口函数、having、group等子句类型
- 子查询不能包含易失函数
- 子查询如果是lateral子句优化需要满足额外的一些条件

#### 优化处理的流程

如果满足上述的子查询优化的条件，后续优化的处理流程由pull_up_simple_subquery函数来完成。

```c
static Node *
pull_up_simple_subquery(PlannerInfo *root, Node *jtnode, RangeTblEntry *rte,
						JoinExpr *lowest_outer_join,
						JoinExpr *lowest_nulling_outer_join,
						AppendRelInfo *containing_appendrel)
{
	Query	   *parse = root->parse;
	int			varno = ((RangeTblRef *) jtnode)->rtindex;
	Query	   *subquery;
	PlannerInfo *subroot;
	int			rtoffset;
	pullup_replace_vars_context rvcontext;
	ListCell   *lc;

	/*
	 * Need a modifiable copy of the subquery to hack on.  Even if we didn't
	 * sometimes choose not to pull up below, we must do this to avoid
	 * problems if the same subquery is referenced from multiple jointree
	 * items (which can't happen normally, but might after rule rewriting).
	 */
	subquery = copyObject(rte->subquery);

	/*
	 * Create a PlannerInfo data structure for this subquery.
	 *
	 * NOTE: the next few steps should match the first processing in
	 * subquery_planner().  Can we refactor to avoid code duplication, or
	 * would that just make things uglier?
	 */
	subroot = makeNode(PlannerInfo);
	subroot->parse = subquery;
	subroot->glob = root->glob;
	subroot->query_level = root->query_level;
	subroot->parent_root = root->parent_root;
	subroot->plan_params = NIL;
	subroot->outer_params = NULL;
	subroot->planner_cxt = CurrentMemoryContext;
	subroot->init_plans = NIL;
	subroot->cte_plan_ids = NIL;
	subroot->multiexpr_params = NIL;
	subroot->eq_classes = NIL;
	subroot->append_rel_list = NIL;
	subroot->rowMarks = NIL;
	memset(subroot->upper_rels, 0, sizeof(subroot->upper_rels));
	memset(subroot->upper_targets, 0, sizeof(subroot->upper_targets));
	subroot->processed_tlist = NIL;
	subroot->grouping_map = NULL;
	subroot->minmax_aggs = NIL;
	subroot->qual_security_level = 0;
	subroot->inhTargetKind = INHKIND_NONE;
	subroot->hasRecursion = false;
	subroot->wt_param_id = -1;
	subroot->non_recursive_path = NULL;

	/* No CTEs to worry about */
	Assert(subquery->cteList == NIL);

	/*
	 * If the FROM clause is empty, replace it with a dummy RTE_RESULT RTE, so
	 * that we don't need so many special cases to deal with that situation.
	 */
	replace_empty_jointree(subquery);

	/*
	 * Pull up any SubLinks within the subquery's quals, so that we don't
	 * leave unoptimized SubLinks behind.
	 */
	if (subquery->hasSubLinks)
		pull_up_sublinks(subroot);

	/*
	 * Similarly, inline any set-returning functions in its rangetable.
	 */
	inline_set_returning_functions(subroot);

	/*
	 * Recursively pull up the subquery's subqueries, so that
	 * pull_up_subqueries' processing is complete for its jointree and
	 * rangetable.
	 *
	 * Note: it's okay that the subquery's recursion starts with NULL for
	 * containing-join info, even if we are within an outer join in the upper
	 * query; the lower query starts with a clean slate for outer-join
	 * semantics.  Likewise, we needn't pass down appendrel state.
	 */
	pull_up_subqueries(subroot);

	/*
	 * Now we must recheck whether the subquery is still simple enough to pull
	 * up.  If not, abandon processing it.
	 *
	 * We don't really need to recheck all the conditions involved, but it's
	 * easier just to keep this "if" looking the same as the one in
	 * pull_up_subqueries_recurse.
	 */
	if (is_simple_subquery(subquery, rte, lowest_outer_join) &&
		(containing_appendrel == NULL || is_safe_append_member(subquery)))
	{
		/* good to go */
	}
	else
	{
		/*
		 * Give up, return unmodified RangeTblRef.
		 *
		 * Note: The work we just did will be redone when the subquery gets
		 * planned on its own.  Perhaps we could avoid that by storing the
		 * modified subquery back into the rangetable, but I'm not gonna risk
		 * it now.
		 */
		return jtnode;
	}

	/*
	 * We must flatten any join alias Vars in the subquery's targetlist,
	 * because pulling up the subquery's subqueries might have changed their
	 * expansions into arbitrary expressions, which could affect
	 * pullup_replace_vars' decisions about whether PlaceHolderVar wrappers
	 * are needed for tlist entries.  (Likely it'd be better to do
	 * flatten_join_alias_vars on the whole query tree at some earlier stage,
	 * maybe even in the rewriter; but for now let's just fix this case here.)
	 */
	subquery->targetList = (List *)
		flatten_join_alias_vars(subroot->parse, (Node *) subquery->targetList);

	/*
	 * Adjust level-0 varnos in subquery so that we can append its rangetable
	 * to upper query's.  We have to fix the subquery's append_rel_list as
	 * well.
	 */
	rtoffset = list_length(parse->rtable);
	OffsetVarNodes((Node *) subquery, rtoffset, 0);
	OffsetVarNodes((Node *) subroot->append_rel_list, rtoffset, 0);

	/*
	 * Upper-level vars in subquery are now one level closer to their parent
	 * than before.
	 */
	IncrementVarSublevelsUp((Node *) subquery, -1, 1);
	IncrementVarSublevelsUp((Node *) subroot->append_rel_list, -1, 1);

	/*
	 * The subquery's targetlist items are now in the appropriate form to
	 * insert into the top query, except that we may need to wrap them in
	 * PlaceHolderVars.  Set up required context data for pullup_replace_vars.
	 */
	rvcontext.root = root;
	rvcontext.targetlist = subquery->targetList;
	rvcontext.target_rte = rte;
	if (rte->lateral)
		rvcontext.relids = get_relids_in_jointree((Node *) subquery->jointree,
												  true);
	else						/* won't need relids */
		rvcontext.relids = NULL;
	rvcontext.outer_hasSubLinks = &parse->hasSubLinks;
	rvcontext.varno = varno;
	/* these flags will be set below, if needed */
	rvcontext.need_phvs = false;
	rvcontext.wrap_non_vars = false;
	/* initialize cache array with indexes 0 .. length(tlist) */
	rvcontext.rv_cache = palloc0((list_length(subquery->targetList) + 1) *
								 sizeof(Node *));

	/*
	 * If we are under an outer join then non-nullable items and lateral
	 * references may have to be turned into PlaceHolderVars.
	 */
	if (lowest_nulling_outer_join != NULL)
		rvcontext.need_phvs = true;

	/*
	 * If we are dealing with an appendrel member then anything that's not a
	 * simple Var has to be turned into a PlaceHolderVar.  We force this to
	 * ensure that what we pull up doesn't get merged into a surrounding
	 * expression during later processing and then fail to match the
	 * expression actually available from the appendrel.
	 */
	if (containing_appendrel != NULL)
	{
		rvcontext.need_phvs = true;
		rvcontext.wrap_non_vars = true;
	}

	/*
	 * If the parent query uses grouping sets, we need a PlaceHolderVar for
	 * anything that's not a simple Var.  Again, this ensures that expressions
	 * retain their separate identity so that they will match grouping set
	 * columns when appropriate.  (It'd be sufficient to wrap values used in
	 * grouping set columns, and do so only in non-aggregated portions of the
	 * tlist and havingQual, but that would require a lot of infrastructure
	 * that pullup_replace_vars hasn't currently got.)
	 */
	if (parse->groupingSets)
	{
		rvcontext.need_phvs = true;
		rvcontext.wrap_non_vars = true;
	}

	/*
	 * Replace all of the top query's references to the subquery's outputs
	 * with copies of the adjusted subtlist items, being careful not to
	 * replace any of the jointree structure. (This'd be a lot cleaner if we
	 * could use query_tree_mutator.)  We have to use PHVs in the targetList,
	 * returningList, and havingQual, since those are certainly above any
	 * outer join.  replace_vars_in_jointree tracks its location in the
	 * jointree and uses PHVs or not appropriately.
	 */
	parse->targetList = (List *)
		pullup_replace_vars((Node *) parse->targetList, &rvcontext);
	parse->returningList = (List *)
		pullup_replace_vars((Node *) parse->returningList, &rvcontext);
	if (parse->onConflict)
	{
		parse->onConflict->onConflictSet = (List *)
			pullup_replace_vars((Node *) parse->onConflict->onConflictSet,
								&rvcontext);
		parse->onConflict->onConflictWhere =
			pullup_replace_vars(parse->onConflict->onConflictWhere,
								&rvcontext);

		/*
		 * We assume ON CONFLICT's arbiterElems, arbiterWhere, exclRelTlist
		 * can't contain any references to a subquery
		 */
	}
	replace_vars_in_jointree((Node *) parse->jointree, &rvcontext,
							 lowest_nulling_outer_join);
	Assert(parse->setOperations == NULL);
	parse->havingQual = pullup_replace_vars(parse->havingQual, &rvcontext);

	/*
	 * Replace references in the translated_vars lists of appendrels. When
	 * pulling up an appendrel member, we do not need PHVs in the list of the
	 * parent appendrel --- there isn't any outer join between. Elsewhere, use
	 * PHVs for safety.  (This analysis could be made tighter but it seems
	 * unlikely to be worth much trouble.)
	 */
	foreach(lc, root->append_rel_list)
	{
		AppendRelInfo *appinfo = (AppendRelInfo *) lfirst(lc);
		bool		save_need_phvs = rvcontext.need_phvs;

		if (appinfo == containing_appendrel)
			rvcontext.need_phvs = false;
		appinfo->translated_vars = (List *)
			pullup_replace_vars((Node *) appinfo->translated_vars, &rvcontext);
		rvcontext.need_phvs = save_need_phvs;
	}

	/*
	 * Replace references in the joinaliasvars lists of join RTEs.
	 *
	 * You might think that we could avoid using PHVs for alias vars of joins
	 * below lowest_nulling_outer_join, but that doesn't work because the
	 * alias vars could be referenced above that join; we need the PHVs to be
	 * present in such references after the alias vars get flattened.  (It
	 * might be worth trying to be smarter here, someday.)
	 */
	foreach(lc, parse->rtable)
	{
		RangeTblEntry *otherrte = (RangeTblEntry *) lfirst(lc);

		if (otherrte->rtekind == RTE_JOIN)
			otherrte->joinaliasvars = (List *)
				pullup_replace_vars((Node *) otherrte->joinaliasvars,
									&rvcontext);
	}

	/*
	 * If the subquery had a LATERAL marker, propagate that to any of its
	 * child RTEs that could possibly now contain lateral cross-references.
	 * The children might or might not contain any actual lateral
	 * cross-references, but we have to mark the pulled-up child RTEs so that
	 * later planner stages will check for such.
	 */
	if (rte->lateral)
	{
		foreach(lc, subquery->rtable)
		{
			RangeTblEntry *child_rte = (RangeTblEntry *) lfirst(lc);

			switch (child_rte->rtekind)
			{
				case RTE_RELATION:
					if (child_rte->tablesample)
						child_rte->lateral = true;
					break;
				case RTE_SUBQUERY:
				case RTE_FUNCTION:
				case RTE_VALUES:
				case RTE_TABLEFUNC:
					child_rte->lateral = true;
					break;
				case RTE_JOIN:
				case RTE_CTE:
				case RTE_NAMEDTUPLESTORE:
				case RTE_RESULT:
					/* these can't contain any lateral references */
					break;
			}
		}
	}

	/*
	 * Now append the adjusted rtable entries to upper query. (We hold off
	 * until after fixing the upper rtable entries; no point in running that
	 * code on the subquery ones too.)
	 */
	parse->rtable = list_concat(parse->rtable, subquery->rtable);

	/*
	 * Pull up any FOR UPDATE/SHARE markers, too.  (OffsetVarNodes already
	 * adjusted the marker rtindexes, so just concat the lists.)
	 */
	parse->rowMarks = list_concat(parse->rowMarks, subquery->rowMarks);

	/*
	 * We also have to fix the relid sets of any PlaceHolderVar nodes in the
	 * parent query.  (This could perhaps be done by pullup_replace_vars(),
	 * but it seems cleaner to use two passes.)  Note in particular that any
	 * PlaceHolderVar nodes just created by pullup_replace_vars() will be
	 * adjusted, so having created them with the subquery's varno is correct.
	 *
	 * Likewise, relids appearing in AppendRelInfo nodes have to be fixed. We
	 * already checked that this won't require introducing multiple subrelids
	 * into the single-slot AppendRelInfo structs.
	 */
	if (parse->hasSubLinks || root->glob->lastPHId != 0 ||
		root->append_rel_list)
	{
		Relids		subrelids;

		subrelids = get_relids_in_jointree((Node *) subquery->jointree, false);
		substitute_phv_relids((Node *) parse, varno, subrelids);
		fix_append_rel_relids(root->append_rel_list, varno, subrelids);
	}

	/*
	 * And now add subquery's AppendRelInfos to our list.
	 */
	root->append_rel_list = list_concat(root->append_rel_list,
										subroot->append_rel_list);

	/*
	 * We don't have to do the equivalent bookkeeping for outer-join info,
	 * because that hasn't been set up yet.  placeholder_list likewise.
	 */
	Assert(root->join_info_list == NIL);
	Assert(subroot->join_info_list == NIL);
	Assert(root->placeholder_list == NIL);
	Assert(subroot->placeholder_list == NIL);

	/*
	 * Miscellaneous housekeeping.
	 *
	 * Although replace_rte_variables() faithfully updated parse->hasSubLinks
	 * if it copied any SubLinks out of the subquery's targetlist, we still
	 * could have SubLinks added to the query in the expressions of FUNCTION
	 * and VALUES RTEs copied up from the subquery.  So it's necessary to copy
	 * subquery->hasSubLinks anyway.  Perhaps this can be improved someday.
	 */
	parse->hasSubLinks |= subquery->hasSubLinks;

	/* If subquery had any RLS conditions, now main query does too */
	parse->hasRowSecurity |= subquery->hasRowSecurity;

	/*
	 * subquery won't be pulled up if it hasAggs, hasWindowFuncs, or
	 * hasTargetSRFs, so no work needed on those flags
	 */

	/*
	 * Return the adjusted subquery jointree to replace the RangeTblRef entry
	 * in parent's jointree; or, if the FromExpr is degenerate, just return
	 * its single member.
	 */
	Assert(IsA(subquery->jointree, FromExpr));
	Assert(subquery->jointree->fromlist != NIL);
	if (subquery->jointree->quals == NULL &&
		list_length(subquery->jointree->fromlist) == 1)
		return (Node *) linitial(subquery->jointree->fromlist);

	return (Node *) subquery->jointree;
}
```

OffsetVarNodes对rtindex进行调整，使RangeTblRef指向原来子查询对应的基表;

IncrementVarSublevelsUp对子查询中Vars所在的层级进行调整；

pullup_replace_vars根据子查询中的变量参数，调整父查询having、returning、targetlist等子句中的变量参数；

完成子查询对应基表上提操作后，使用子查询的jointree代替原来的RangeTblRef。

#### 优化实例

```sql
-- 优化SQL
test=# select * from st,(select * from ss where id<3) as sub where st.id=sub.id and st.id in (select id from tmp where id <3);

-- 执行计划
test=# explain analyze select * from st,(select * from ss where id<3) as sub where st.id=sub.id and st.id in (select id from tmp where id <3);
                                                  QUERY PLAN                                                   
---------------------------------------------------------------------------------------------------------------
 Nested Loop Semi Join  (cost=1.11..29.96 rows=1 width=50) (actual time=0.147..0.166 rows=2 loops=1)
   Join Filter: (st.id = tmp.id)
   Rows Removed by Join Filter: 1
   ->  Hash Join  (cost=1.11..28.68 rows=11 width=50) (actual time=0.122..0.130 rows=2 loops=1)
         Hash Cond: (ss.id = st.id)
         ->  Seq Scan on ss  (cost=0.00..25.88 rows=423 width=36) (actual time=0.069..0.073 rows=2 loops=1)
               Filter: (id < 3)
               Rows Removed by Filter: 1
         ->  Hash  (cost=1.05..1.05 rows=5 width=14) (actual time=0.026..0.027 rows=5 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 9kB
               ->  Seq Scan on st  (cost=0.00..1.05 rows=5 width=14) (actual time=0.013..0.017 rows=5 loops=1)
   ->  Materialize  (cost=0.00..1.12 rows=1 width=4) (actual time=0.008..0.009 rows=2 loops=2)
         ->  Seq Scan on tmp  (cost=0.00..1.11 rows=1 width=4) (actual time=0.009..0.011 rows=2 loops=1)
               Filter: (id < 3)
 Planning Time: 1.455 ms
 Execution Time: 0.357 ms
(16 rows)
```

#### stack

![img](http://chuantu.xyz/t6/738/1592129493x3661913030.png) 

#### 优化前jointree

![img](http://chuantu.xyz/t6/738/1592129597x2728290871.png) 

#### 子链接优化后的jointree

![img](http://chuantu.xyz/t6/738/1592129667x2728290871.png) 

#### 子链接优化后的rtable

![img](http://chuantu.xyz/t6/738/1592129928x2728290871.png) 

#### 子查询优化后的jointree

![img](http://chuantu.xyz/t6/738/1592129733x2728290871.png) 

#### 子查询优化后的rtable

![img](http://chuantu.xyz/t6/738/1592129973x2728290871.png) 