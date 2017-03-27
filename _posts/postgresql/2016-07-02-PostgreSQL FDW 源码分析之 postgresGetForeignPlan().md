---
layout: post
title:  "PostgreSQL FDW 源码分析之 postgresGetForeignPlan()"
date:   2016-07-02 22:33 +0800
categories: postgresql  
---

进入 postgresGetForeignPlan()

根据在 GetForeignRelSize() 中处理获得的 attrs_used 构造要发送到 remote 端的 SQL 语句，其中包括 targetList 和 local_conds 中提到的属性，然后加上属于 remote_conds 的 where 限制条件。

```
	/*
	 * Build the query string to be sent for execution, and identify
	 * expressions to be sent as parameters.
	 */
	initStringInfo(&sql);
	deparseSelectSql(&sql, root, baserel, fpinfo->attrs_used,
					 &retrieved_attrs);
	if (remote_conds)
		appendWhereClause(&sql, root, baserel, remote_conds,
						  true, ¶ms_list);
```

```
	/*
	 * Build the fdw_private list that will be available to the executor.
	 * Items in the list must match enum FdwScanPrivateIndex, above.
	 */
	fdw_private = list_make2(makeString(sql.data),
				 retrieved_attrs);
	/*
	 * Create the ForeignScan node from target list, local filtering
	 * expressions, remote parameter expressions, and FDW private information.
	 *
	 * Note that the remote parameter expressions are stored in the fdw_exprs
	 * field of the finished plan node; we can't keep them in private state
	 * because then they wouldn't be subject to later planner processing.
	 */
	return make_foreignscan(tlist,
							local_exprs,
							scan_relid,
							params_list,
							fdw_private,
							NIL,	/* no custom tlist */
							remote_exprs,
							outer_plan);
```