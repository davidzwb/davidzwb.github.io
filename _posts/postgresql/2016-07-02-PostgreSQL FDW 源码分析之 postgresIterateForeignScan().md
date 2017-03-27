---
layout: post
title:  "PostgreSQL FDW 源码分析之 postgresIterateForeignScan()"
date:   2016-07-02 22:34 +0800
categories: postgresql  
---

进入 postgresIterateForeignScan()

在 fsstate 中保存有之前处理的信息，包括要发向 remote 端的 SQL 语句：

```
	PgFdwScanState *fsstate = (PgFdwScanState *) node->fdw_state;
	TupleTableSlot *slot = node->ss.ss_ScanTupleSlot;
```

```
/*
 * If this is the first call after Begin or ReScan, we need to create the
 * cursor on the remote side.
 */
if (!fsstate->cursor_exists)
  create_cursor(node);
```

这里构造了一个创建游标的 SQL 语句，DECLARE c1 CURSOR FOR “fsstate->query”，其中 “fsstate->query” 就是存放在 fsstate 中的、之前构造的那个 SQL 语句。

```
/* Construct the DECLARE CURSOR command */
initStringInfo(&buf);
appendStringInfo(&buf, "DECLARE c%u CURSOR FOR\n%s",
         fsstate->cursor_number, fsstate->query);
```

此处发送到 remote 端执行，并将返回的结果保存到 PGresult 类型的 res 中； 

```
res = PQexecParams(conn, buf.data, numParams, NULL, values,
           NULL, NULL, 0);
```

然后在下面的 fetch_more_data() 中，进行数据读取：

```
	/*
	 * Get some more tuples, if we've run out.
	 */
	if (fsstate->next_tuple >= fsstate->num_tuples)
	{
		/* No point in another fetch if we already detected EOF, though. */
		if (!fsstate->eof_reached)
			fetch_more_data(node);
		/* If we didn't get any tuples, must be end of data. */
		if (fsstate->next_tuple >= fsstate->num_tuples)
			return ExecClearTuple(slot);
	}
```

```
		/* The fetch size is arbitrary, but shouldn't be enormous. */
		fetch_size = 100;

		snprintf(sql, sizeof(sql), "FETCH %d FROM c%u",
				 fetch_size, fsstate->cursor_number);

		res = PQexec(conn, sql);
	/*
	 * Return the next tuple.
	 */
	ExecStoreTuple(fsstate->tuples[fsstate->next_tuple++],
				   slot,
				   InvalidBuffer,
				   false);
```