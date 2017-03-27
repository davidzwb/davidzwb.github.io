---
layout: post
title:  "PostgreSQL FDW 源码分析之 GetForeignRelSize()"
date:   2016-06-29 09:54 +0800
categories: postgresql  
---

GetForeignRelSize() 的源码在 postgres_fdw.c 文件中。

进入 GetForeignRelSize() 一开头，将 FDW 相关的 planner 信息存放在 PgFdwRelationInfo *fpinfo 中。

```
PgFdwRelationInfo *fpinfo;

fpinfo = (PgFdwRelationInfo *) palloc0(sizeof(PgFdwRelationInfo));
baserel->fdw_private = (void *) fpinfo;
```

```
/*
 * FDW-specific planner information kept in RelOptInfo.fdw_private for a
 * foreign table.  This information is collected by postgresGetForeignRelSize.
 */
typedef struct PgFdwRelationInfo
{
	/* baserestrictinfo clauses, broken down into safe and unsafe subsets. */
	List	   *remote_conds;
	List	   *local_conds;

	/* Bitmap of attr numbers we need to fetch from the remote server. */
	Bitmapset  *attrs_used;

	/* Cost and selectivity of local_conds. */
	QualCost	local_conds_cost;
	Selectivity local_conds_sel;

	/* Estimated size and cost for a scan with baserestrictinfo quals. */
	double		rows;
	int		width;
	Cost		startup_cost;
	Cost		total_cost;

	/* Options extracted from catalogs. */
	bool		use_remote_estimate;
	Cost		fdw_startup_cost;
	Cost		fdw_tuple_cost;

	/* Cached catalog information. */
	ForeignTable *table;
	ForeignServer *server;
	UserMapping *user;			/* only set in use_remote_estimate mode */
} PgFdwRelationInfo;

```

```
/* Look up foreign-table catalog info. */
fpinfo->table = GetForeignTable(foreigntableid);
fpinfo->server = GetForeignServer(fpinfo->table->serverid);
```

```
fpinfo->use_remote_estimate = false;
fpinfo->fdw_startup_cost = DEFAULT_FDW_STARTUP_COST;
fpinfo->fdw_tuple_cost = DEFAULT_FDW_TUPLE_COST;
```

若保持默认 use_remote_estimate = false，在下面的 classifyConditions() 中，对输入的 SQL 语句中的 where 限制条件进行逐项归类，将之分为 remote_conds 和 local_conds，前者可以发送到 remote 端处理，后者不能。

```
void
classifyConditions(PlannerInfo *root,
				   RelOptInfo *baserel,
				   List *input_conds,
				   List **remote_conds,
				   List **local_conds)
{
	ListCell   *lc;

	*remote_conds = NIL;
	*local_conds = NIL;

	foreach(lc, input_conds)
	{
		RestrictInfo *ri = (RestrictInfo *) lfirst(lc);

		if (is_foreign_expr(root, baserel, ri->clause))
			*remote_conds = lappend(*remote_conds, ri);
		else
			*local_conds = lappend(*local_conds, ri);
	}
}
```

\1. 在 foreign_expr_walker() 中对限制条件进行判断，只有 built-in 类型的才能发送到 remote 端，因为非 built-in 的类型在 remote 端不能保证支持。但如果需要，应该可以修改源码加入对自定义类型的支持。

\2. 在 contain_mutable_functions() 中判断是否是 mutable function，只有非 mutable function 才能发送到 remote 端。（如注释所述，For example, sending now() to remote side could cause confusion from clock offsets.）

总结：只有内置类型或非 mutable 函数类型的限制条件才会发送到 remote 端进行处理。

```
bool
is_foreign_expr(PlannerInfo *root,
				RelOptInfo *baserel,
				Expr *expr)
{
	foreign_glob_cxt glob_cxt;
	foreign_loc_cxt loc_cxt;

	/*
	 * Check that the expression consists of nodes that are safe to execute
	 * remotely.
	 */
	glob_cxt.root = root;
	glob_cxt.foreignrel = baserel;
	loc_cxt.collation = InvalidOid;
	loc_cxt.state = FDW_COLLATE_NONE;
	if (!foreign_expr_walker((Node *) expr, &glob_cxt, &loc_cxt))
		return false;

	/* Expressions examined here should be boolean, ie noncollatable */
	Assert(loc_cxt.collation == InvalidOid);
	Assert(loc_cxt.state == FDW_COLLATE_NONE);

	/*
	 * An expression which includes any mutable functions can't be sent over
	 * because its result is not stable.  For example, sending now() remote
	 * side could cause confusion from clock offsets.  Future versions might
	 * be able to make this choice with more granularity.  (We check this last
	 * because it requires a lot of expensive catalog lookups.)
	 */
	if (contain_mutable_functions((Node *) expr))
		return false;

	/* OK to evaluate on the remote server */
	return true;
}
```

然后对需要从 remote 端取回的表属性进行标记，需要取回的表属性包括 targetList ( 即select attr from table; 中的 attr) 中的属性和限制条件中本地处理的属性：

attrs_used 在之后构造发送到 remote 端的 SQL 语句时，起主要作用。

```
	/*
	 * Identify which attributes will need to be retrieved from the remote
	 * server.  These include all attrs needed for joins or final output, plus
	 * all attrs used in the local_conds.  (Note: if we end up using a
	 * parameterized scan, it's possible that some of the join clauses will be
	 * sent to the remote and thus we wouldn't really need to retrieve the
	 * columns used in them.  Doesn't seem worth detecting that case though.)
	 */
	fpinfo->attrs_used = NULL;
	pull_varattnos((Node *) baserel->reltargetlist, baserel->relid,
				   &fpinfo->attrs_used);
	foreach(lc, fpinfo->local_conds)
	{
		RestrictInfo *rinfo = (RestrictInfo *) lfirst(lc);

		pull_varattnos((Node *) rinfo->clause, baserel->relid,
					   &fpinfo->attrs_used);
	}
```

```
	/*
	 * Compute the selectivity and cost of the local_conds, so we don't have
	 * to do it over again for each path.  The best we can do for these
	 * conditions is to estimate selectivity on the basis of local statistics.
	 */
	fpinfo->local_conds_sel = clauselist_selectivity(root,fpinfo->local_conds,baserel->relid,JOIN_INNER,NULL);

	cost_qual_eval(&fpinfo->local_conds_cost, fpinfo->local_conds, root);
```

然后，基于本地数据预估表大小，对于外部表，则假定表占用的空间为 10 页，来计算：

```
		/*
		 * If the foreign table has never been ANALYZEd, it will have relpages
		 * and reltuples equal to zero, which most likely has nothing to do
		 * with reality.  We can't do a whole lot about that if we're not
		 * allowed to consult the remote server, but we can use a hack similar
		 * to plancat.c's treatment of empty relations: use a minimum size
		 * estimate of 10 pages, and divide by the column-datatype-based width
		 * estimate to get the corresponding number of tuples.
		 */
		if (baserel->pages == 0 && baserel->tuples == 0)
		{
			baserel->pages = 10;
			baserel->tuples =
				(10 * BLCKSZ) / (baserel->width + MAXALIGN(SizeofHeapTupleHeader));
		}

		/* Estimate baserel size as best we can with local statistics. */
		set_baserel_size_estimates(root, baserel);

```

若 use_remote_estimate = true，那么会构造一个 EXPLAIN 的 SQL 语句，构造方式如下，其中包含之前筛选出的适合发送到 remote 端处理的 remote_conds，发送到 remote 端处理估算，然后发回。

```
		/*
		 * Construct EXPLAIN query including the desired SELECT, FROM, and
		 * WHERE clauses.  Params and other-relation Vars are replaced by
		 * dummy values.
		 */
		initStringInfo(&sql);
		appendStringInfoString(&sql, "EXPLAIN ");
		deparseSelectSql(&sql, root, baserel, fpinfo->attrs_used,
						 &retrieved_attrs);
		if (fpinfo->remote_conds)
			appendWhereClause(&sql, root, baserel, fpinfo->remote_conds,
							  true, NULL);
		if (remote_join_conds)
			appendWhereClause(&sql, root, baserel, remote_join_conds,
							  (fpinfo->remote_conds == NIL), NULL);
```