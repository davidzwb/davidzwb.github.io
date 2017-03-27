---
layout: post
title:  "PostgreSQL 优化器中 Window 解析处理"
date:   2016-11-30 17:35 +0800
categories: postgresql 
---
# PostgreSQL 优化器中 Window 解析处理

对于如下的表：

```sql
CREATE TABLE empsalary(
  depname varchar,
  empno bigint,
  salary int,
  enroll_date date
);
```

对于如下这个语句：

```sql
select rank() OVER (PARTITION BY depname ORDER BY salary), * from empsalary;
```

PG 需要如下的参数来创建 Window 语句的计划路径：

```c
create_window_paths(root, current_rel, grouping_target, final_target, tlist, wflists, activeWindows);
```

## PG 首先做的事是生成两个 PathTarget 结构：

```c
/*
* Convert the query's result tlist into PathTarget format.
*/
final_target = create_pathtarget(root, tlist);
```

>p *(PathTarget *)final_target     
>$17 = {type = T_PathTarget, exprs = 0x2ca7408, sortgrouprefs = 0x2ca7540, cost = {startup = 0, per_tuple = 0}, width = 48}

final_target 中的 exprs 为一个 List 链表，包含 4 个 Var 元素：

> {xpr = {type = T_Var}, varno = 1, varattno = 1, vartype = 1043, vartypmod = -1, varcollid = 100, varlevelsup = 0, varnoold = 1, varoattno = 1, location = 59}
>
> {xpr = {type = T_Var}, varno = 1, varattno = 3, vartype = 23, vartypmod = -1, varcollid = 0, varlevelsup = 0, varnoold = 1, varoattno = 3, location = 59}
>
> {xpr = {type = T_Var}, varno = 1, varattno = 2, vartype = 20, vartypmod = -1, varcollid = 0, varlevelsup = 0, varnoold = 1, varoattno = 2, location = 59}
>
> {xpr = {type = T_Var}, varno = 1, varattno = 4, vartype = 1082, vartypmod = -1, varcollid = 0, varlevelsup = 0, varnoold = 1, varoattno = 4, location = 59}

它们其实就是 empsalary 表的 4 个属性，即 SQL 查询语句中的 '*'。

而 final_target 中的 sortgrouprefs 是一个 Index 类型的数组：

>(gdb) p *((Index *)0x2ca7540)@4
>
>$44 = {2, 1, 0, 0}

它们的作用要之后才能够看出。

## 然后生成另一个 PathTarget:

```c
if (activeWindows)
			grouping_target = make_window_input_target(root,
													   final_target,
													   activeWindows);
```

>p *(PathTarget *)grouping_target
>
>$29 = {type = T_PathTarget, exprs = 0x2ca7280, sortgrouprefs = 0x2ca7230, cost = {startup = 0, per_tuple = 0}, width = 56}

grouping_target 中的 exprs 为一个 List 链表，包含 5 个元素：

>{xpr = {type = T_WindowFunc}, winfnoid = 3101, wintype = 20, wincollid = 0, inputcollid = 0, args = 0x0, aggfilter = 0x0, winref = 1, winstar = 0 '\000',  winagg = 0 '\000', location = 7}
>
>{xpr = {type = T_Var}, varno = 1, varattno = 1, vartype = 1043, vartypmod = -1, varcollid = 100, varlevelsup = 0, varnoold = 1, varoattno = 1, location = 59}
>
>{xpr = {type = T_Var}, varno = 1, varattno = 3, vartype = 23, vartypmod = -1, varcollid = 0, varlevelsup = 0, varnoold = 1, varoattno = 3, location = 59}
>
>{xpr = {type = T_Var}, varno = 1, varattno = 2, vartype = 20, vartypmod = -1, varcollid = 0, varlevelsup = 0, varnoold = 1, varoattno = 2, location = 59}
>
>{xpr = {type = T_Var}, varno = 1, varattno = 4, vartype = 1082, vartypmod = -1, varcollid = 0, varlevelsup = 0, varnoold = 1, varoattno = 4, location = 59}

可以看到其实就是在其中增加了 WindowFunc 项，它也就是 SQL 查询语句中的 rank()。

而 grouping_target 中的 sortgrouprefs 是一个 Index 类型的数组：

>p *((Index *)0x2ca7230)@5
>
>$60 = {0, 2, 0, 1, 0}

这样 PG 就得到了两个 Target 信息。

## 生成 wflist  - targetlist 中窗口函数的信息：

```c
/*
		 * Locate any window functions in the tlist.  (We don't need to look
		 * anywhere else, since expressions used in ORDER BY will be in there
		 * too.)  Note that they could all have been eliminated by constant
		 * folding, in which case we don't need to do any more work.
		 */
if (parse->hasWindowFuncs)
{
  wflists = find_window_functions((Node *) tlist,
                                  list_length(parse->windowClause));
  if (wflists->numWindowFuncs > 0)
    activeWindows = select_active_windows(root, wflists);
  else
    parse->hasWindowFuncs = false;
}
```

生成 wflist 的方式是从 tlist (tlist 即经过一些预操作的 targetlist) 中筛选出的窗口函数部分的信息：

>p *(WindowFuncLists *)wflists 
>$48 = {numWindowFuncs = 1, maxWinRef = 1, windowFuncs = 0x2ca61a8}

## 生成 activeWindows： 

```c
if (parse->hasWindowFuncs)
{
  wflists = find_window_functions((Node *) tlist,
                                  list_length(parse->windowClause));
  if (wflists->numWindowFuncs > 0)
    activeWindows = select_active_windows(root, wflists);
  else
    parse->hasWindowFuncs = false;
}
```

将 QueryTree 中 active 的 WindowClause 放到一个链表中。

>activeWindows:List:
>{type = T_WindowClause, name = 0x0, refname = 0x0, partitionClause = 0x2ca5900, orderClause = 0x2ca57e0, frameOptions = 530, startOffset = 0x0, endOffset = 0x0, winref = 1, copiedOrder = 0 '\000'}
>
>
>partitionClause:List:
>{type = T_SortGroupClause, tleSortGroupRef = 2, eqop = 98, sortop = 664, nulls_first = 0 '\000', hashable = 1 '\001'}
>
>orderClause:List:
>{type = T_SortGroupClause, tleSortGroupRef = 1, eqop = 96, sortop = 97, nulls_first = 0 '\000', hashable = 1 '\001'}

可以看到 partitionClause 和 orderClause 中的 tleSortGroupRef 其实就是对应的 PathTarget 中的 sortgrouprefs；

在 partitionClause 中 tleSortGroupRef = 2，在 final_target 中的 sortgrouprefs 数组里，第 0 项的值也是 2，而 第 0 项对应的是 final_target 中 expr list 中的第 0 个，该 Var 结构对应的则是 attribute number = 1 的 empsalary 表中的 depname。

orderClause 也是同样的道理。

## 开始生成 Window 的计划路径 Path：

PG 调用如下函数，

```c
/*
		 * If we have window functions, consider ways to implement those.  We
		 * build a new upperrel representing the output of this phase.
		 */
if (activeWindows)
{
  current_rel = create_window_paths(root,
                                    current_rel,
                                    grouping_target,
                                    final_target,
                                    tlist,
                                    wflists,
                                    activeWindows);
}
```

其中 root 为 PlannerInfo，current_rel 为之前通过 query_planner() 生成的基本或关联表的 RelOptInfo。

在 create_window_paths() 中：

```c
/* For now, do all work in the (WINDOW, NULL) upperrel */
window_rel = fetch_upper_rel(root, UPPERREL_WINDOW, NULL);
```

申请一个 window 类型的 upper rel。

为每一个现存的 path 增加 window path 以及 sort path

```c
	/*
	 * Consider computing window functions starting from the existing
	 * cheapest-total path (which will likely require a sort) as well as any
	 * existing paths that satisfy root->window_pathkeys (which won't).
	 */
foreach(lc, input_rel->pathlist)
{
  Path	   *path = (Path *) lfirst(lc);

  if (path == input_rel->cheapest_total_path ||
      pathkeys_contained_in(root->window_pathkeys, path->pathkeys))
    create_one_window_path(root,
                           window_rel,
                           path,
                           input_target,
                           output_target,
                           tlist,
                           wflists,
                           activeWindows);
}
```

最后生成的 pathlist 是这样的，WindowAggPath->SortPath->Path 如下：

```c
{path = {type = T_WindowAggPath, pathtype = T_WindowAgg, parent = 0x2cb2ff0, pathtarget = 0x2ca71e0, param_info = 0x0, parallel_aware = 0 '\000', parallel_safe = 0 '\000', parallel_workers = 0, rows = 1070, startup_cost = 74.539163684893566, total_cost = 95.939163684893558, pathkeys = 0x2cb32b0}, subpath = 0x2cb3368, winclause = 0x2ca5930, winpathkeys = 0x2cb32b0}

{path = {type = T_SortPath, pathtype = T_Sort, parent = 0x2cb2ff0, pathtarget = 0x2ca7348, param_info = 0x0, parallel_aware = 0 '\000', parallel_safe = 0 '\000', parallel_workers = 0, rows = 1070, startup_cost = 74.539163684893566, total_cost = 77.214163684893563, pathkeys = 0x2cb32b0}, subpath = 0x2ca70d0}

{type = T_Path, pathtype = T_SeqScan, parent = 0x2ca62a8, pathtarget = 0x2ca7348, param_info = 0x0, parallel_aware = 0 '\000', parallel_safe = 0 '\000', parallel_workers = 0, rows = 1070, startup_cost = 0, total_cost = 20.700000000000003, pathkeys = 0x0}
```

