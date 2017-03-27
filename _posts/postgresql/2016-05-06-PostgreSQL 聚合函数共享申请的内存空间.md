---
layout: post
title:  "PostgreSQL 聚合函数共享申请的内存空间"
date:   2016-05-06 16:10 +0800
categories: postgresql 
---

```
CREATE AGGREGATE union(bitmap)
(
    sfunc = myfunction,
    stype = mytype,
    FINALFUNC = myfunction_final
);
```

在编写聚合函数时，对每一行都会重复调用指定同一函数，如果要处理的数据是累加的，那么如果不在每次调用之间共享内存空间，而是不停的申请释放新的内存，那么速度会变得很慢，所以在这时共享内存是十分有用的：

PostgreSQL 有 MemoryContext 的概念，如果普通的使用 palloc 申请内存空间，系统会向 CurrentMemoryContext 申请，而据我试验猜测，聚合函数在每次调用时，都会切换 CurrentMemoryContext，所以普通的 palloc 是不能使用的。

```
#define PG_FUNCTION_ARGS	FunctionCallInfo fcinfo
```

FunctionCallInfo 是指向 FunctionCallInfoData 结构的指针：

```
/*
 * This struct is the data actually passed to an fmgr-called function.
 */
typedef struct FunctionCallInfoData
{
	FmgrInfo   *flinfo;			/* ptr to lookup info used for this call */
	fmNodePtr	context;		/* pass info about context of call */
	fmNodePtr	resultinfo;		/* pass or return extra info about result */
	Oid			fncollation;	/* collation for function to use */
	bool		isnull;			/* function must set true if result is NULL */
	short		nargs;			/* # arguments actually passed */
	Datum		arg[FUNC_MAX_ARGS];		/* Arguments passed to function */
	bool		argnull[FUNC_MAX_ARGS]; /* T if arg[i] is actually NULL */
} FunctionCallInfoData;
```

```
typedef struct FmgrInfo
{
	PGFunction	fn_addr;		/* pointer to function or handler to be called */
	Oid		fn_oid;			/* OID of function (NOT of handler, if any) */
	short		fn_nargs;		/* number of input args (0..FUNC_MAX_ARGS) */
	bool		fn_strict;		/* function is "strict" (NULL in => NULL out) */
	bool		fn_retset;		/* function returns a set */
	unsigned char fn_stats;		/* collect stats if track_functions > this */
	<span style="color:#ff0000;">void	   *fn_extra</span>;		/* extra space for use by handler */
	<span style="color:#ff0000;">MemoryContext fn_mcxt</span>;		/* memory context to store fn_extra in */
	fmNodePtr	fn_expr;		/* expression parse tree for call, or NULL */
} FmgrInfo;
```

向指定 MemoryContext - fn_mcxt 申请内存的函数如下：

```
MemoryContextAlloc(fcinfo->flinfo->fn_mcxt, sizeof(some_type));
```

可以参考 `src/backend/utils/adt/arrayfuncs.c 以及下列文章。`

参考文章：

[http://stackoverflow.com/questions/30515552/can-a-postgres-c-language-function-reference-a-stateful-variable-c-side-possibl](http://stackoverflow.com/questions/30515552/can-a-postgres-c-language-function-reference-a-stateful-variable-c-side-possibl)