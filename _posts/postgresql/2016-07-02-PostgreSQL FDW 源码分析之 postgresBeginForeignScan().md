---
layout: post
title:  "PostgreSQL FDW 源码分析之 postgresBeginForeignScan()"
date:   2016-07-02 11:12 14:27 +0800
categories: postgresql  
---

在 postgresBeginForeignScan() 中主要进行 postgresIterateForeignScan() 中需要数据的准备工作。

获取建立的外部表信息：

```
	/* Get info about foreign table. */
	fsstate->rel = node->ss.ss_currentRelation;
	table = GetForeignTable(RelationGetRelid(fsstate->rel));
	server = GetForeignServer(table->serverid);
	user = GetUserMapping(userid, server->serverid);
```

```
	/*
	 * Get connection to the foreign server.  Connection manager will
	 * establish new connection if necessary.
	 */
	fsstate->conn = GetConnection(server, user, false);

	/* Assign a unique ID for my cursor */
	fsstate->cursor_number = GetCursorNumber(fsstate->conn);
	fsstate->cursor_exists = false;
```

```
	/* Get private info created by planner functions. */
	fsstate->query = strVal(list_nth(fsplan->fdw_private,
									 FdwScanPrivateSelectSql));
	fsstate->retrieved_attrs = (List *) list_nth(fsplan->fdw_private,
											   FdwScanPrivateRetrievedAttrs);
	/* Create contexts for batches of tuples and per-tuple temp workspace. */
	fsstate->batch_cxt = AllocSetContextCreate(estate->es_query_cxt,
											   "postgres_fdw tuple data",
											   ALLOCSET_DEFAULT_MINSIZE,
											   ALLOCSET_DEFAULT_INITSIZE,
											   ALLOCSET_DEFAULT_MAXSIZE);
	fsstate->temp_cxt = AllocSetContextCreate(estate->es_query_cxt,
											  "postgres_fdw temporary data",
											  ALLOCSET_SMALL_MINSIZE,
											  ALLOCSET_SMALL_INITSIZE,
											  ALLOCSET_SMALL_MAXSIZE);
```