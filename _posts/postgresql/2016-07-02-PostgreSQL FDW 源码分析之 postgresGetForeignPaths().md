---
layout: post
title:  "PostgreSQL FDW 源码分析之 postgresGetForeignPaths()"
date:   2016-07-02 14:57 +0800
categories: postgresql  
---

进入 postgresGetForeignPaths()

在 postgresGetForeignRelSize() 中已经对最基本的外部表扫描做了成本估算，所以就用这些信息，直接生成一个 plan path：

```
/*
 * Create simplest ForeignScan path node and add it to baserel.  This path
 * corresponds to SeqScan path of regular tables (though depending on what
 * baserestrict conditions we were able to send to remote, there might
 * actually be an indexscan happening there).  We already did all the work
 * to estimate cost and size of this path.
 */
path = create_foreignscan_path(root, baserel,
                 fpinfo->rows,
                 fpinfo->startup_cost,
                 fpinfo->total_cost,
                 NIL, /* no pathkeys */
                 NULL,		/* no outer rel either */
                 NULL,		/* no extra plan */
                 NIL);		/* no fdw_private list */
```

```
add_path(baserel, (Path *) path);
```

```
	/*
	 * If we're not using remote estimates, stop here.  We have no way to
	 * estimate whether any join clauses would be worth sending across, so
	 * don't bother building parameterized paths.
	 */
	if (!fpinfo->use_remote_estimate)
		return;
```

join path 的计算结果存放在 param_info 中，并挂在 ppi_list 的链表上。

```
/* Get the ParamPathInfo */
		param_info = get_baserel_parampathinfo(root, baserel,
											   required_outer);
		Assert(param_info != NULL);

		/*
		 * Add it to list unless we already have it.  Testing pointer equality
		 * is OK since get_baserel_parampathinfo won't make duplicates.
		 */
		ppi_list = list_append_unique_ptr(ppi_list, param_info);
```

每一种 join 的情况，都对应一个挂在 ppi_list 上的 param_info，对于每一个 param_info 都生成一个 path，与上面的基本扫描相比，它们都带上了 param_info->ppi_req_outer。

```
foreach(lc, ppi_list)
{
  ParamPathInfo *param_info = (ParamPathInfo *) lfirst(lc);
  double		rows;
  int			width;
  Cost		startup_cost;
  Cost		total_cost;

  /* Get a cost estimate from the remote */
  estimate_path_cost_size(root, baserel,
              param_info->ppi_clauses,
              &rows, &width,
              &startup_cost, &total_cost);

  /*
   * ppi_rows currently won't get looked at by anything, but still we
   * may as well ensure that it matches our idea of the rowcount.
   */
  param_info->ppi_rows = rows;

  /* Make the path */
  path = create_foreignscan_path(root, baserel,
                   rows,
                   startup_cost,
                   total_cost,
                   NIL,		/* no pathkeys */
                   param_info->ppi_req_outer,
                   NULL,
                   NIL);	/* no fdw_private list */
  add_path(baserel, (Path *) path);
}
```

scan_clauses 是准备发送给执行器执行语句的限制条件，这里如果存在 best_path->param_info 的话，就会将其中带有的 join 限制条件也加入 scan_clauses。

```
/*
 * Extract the relevant restriction clauses from the parent relation. The
 * executor must apply all these restrictions during the scan, except for
 * pseudoconstants which we'll take care of below.
 */
scan_clauses = rel->baserestrictinfo;

/*
 * If this is a parameterized scan, we also need to enforce all the join
 * clauses available from the outer relation(s).
 *
 * For paranoia's sake, don't modify the stored baserestrictinfo list.
 */
if (best_path->param_info)
  scan_clauses = list_concat(list_copy(scan_clauses),
                 best_path->param_info->ppi_clauses);
```