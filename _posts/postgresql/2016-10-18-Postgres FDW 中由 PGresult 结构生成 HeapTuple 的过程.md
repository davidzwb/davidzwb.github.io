---
layout: post
title:  "Postgres FDW 中由 PGresult 结构生成 HeapTuple 的过程"
date:   2016-10-18 17:34 +0800
categories: postgresql  
---

### PGresult 结构：

```
struct pg_result
{
	int			ntups;
	int			numAttributes;
	PGresAttDesc *attDescs;
	PGresAttValue **tuples;
	int			tupArrSize;	
	int			numParameters;
	PGresParamDesc *paramDescs;
	ExecStatusType resultStatus;
	char		cmdStatus[CMDSTATUS_LEN];


	PGNoticeHooks noticeHooks;
	PGEvent    *events;
	int			nEvents;
	int			client_encoding;	/* encoding id */


	char	   *errMsg;			/* error message, or NULL if no error */
	PGMessageField *errFields;	/* message broken into fields */
	char	   *errQuery;		/* text of triggering query, if available */


	PGresult_data *curBlock;	/* most recently allocated block */
	int			curOffset;		/* start offset of free space in block */
	int			spaceLeft;		/* number of free bytes remaining in block */
};
```

### **PGresult.tuples：**

返回的 tuple 的实际的值都存放在 tuples 中，tuples 是一个 PGresAttValue * 类型的二位数组，
tuples[row_number][column_number] 就代表返回的表(关系)中第 row_number 行第 column_number 列中数据的值；

### PGresult.PGresAttDesc:

```
typedef struct pgresAttDesc
{
	char	   *name;			/* column name */
	Oid			tableid;		/* source table, if known */
	int			columnid;		/* source column, if known */
	int			format;			/* format code for value (text/binary) */
	Oid			typid;			/* type id */
	int			typlen;			/* type size */
	int			atttypmod;		/* type-specific modifier info */
} PGresAttDesc;
```

从远端取回的 PGresult 一定会对应一个本地已经生成的表，可能是一张基本表或是一张由 join 形成的临时的表，通过下面的语句获取该表的 tupleDesc 元组描述信息：

```
	if (rel)
		tupdesc = RelationGetDescr(rel);
	else
	{
		PgFdwScanState *fdw_sstate;

		Assert(fsstate);
		fdw_sstate = (PgFdwScanState *) fsstate->fdw_state;
		tupdesc = fdw_sstate->tupdesc;
	}
```
由于 FDW 会对发到远端执行的语句有筛选，所以函数还传入了一个 retrieved_attrs，retrieved_attrs 是一个 List，每一项表示从远端取回的列(属性)的列 id，从 1 开始计数，

```
	foreach(lc, retrieved_attrs)
	{
		int	   i = lfirst_int(lc);
		char	   *valstr;

		/* fetch next column's textual value */
		if (PQgetisnull(res, row, j))
			valstr = NULL;
		else
			valstr = PQgetvalue(res, row, j);

		/* convert value to internal representation */
		errpos.cur_attno = i;
		if (i > 0)
		{
			/* ordinary column */
			Assert(i <= tupdesc->natts);
			nulls[i - 1] = (valstr == NULL);
			/* Apply the input function even to nulls, to support domains */
			values[i - 1] = InputFunctionCall(&attinmeta->attinfuncs[i - 1],
											  valstr,
											  attinmeta->attioparams[i - 1],
											  attinmeta->atttypmods[i - 1]);
		}
```

在 bool nulls[] 数组中存储每一列中是否为空，值为 true(0) 或 false(1)，

在 datum values[] 中则存放由 C 字符串转化为 tuple datum 格式的数据值，转化的方法由 attinmeta 提供。

上述的 tupleDesc, retrieved_attrs 和 attinmeta 若有需要，都是可以通过调用一些函数自己构造的，文末将会提供构造方法。
最后调用 heap_form_tuple(tupdesc, values, nulls) 由 tupdesc,、values[]、nulls[] 构造一个 HeapTuple。 