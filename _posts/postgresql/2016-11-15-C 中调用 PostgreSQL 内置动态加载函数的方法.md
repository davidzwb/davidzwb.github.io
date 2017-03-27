---
layout: post
title:  "C 中调用 PostgreSQL 内置动态加载函数的方法"
date:   2016-11-15 16:09 +0800
categories: postgresql  
---
# C 中调用 PostgreSQL 内置动态加载函数的方法 

## 普通函数

### 使用位于 fmgr.h 中的各类 DirectFunctionCall 函数

比如若想将一个时间间隔字符串转换为一个 PG 内置 interval 类型，PG 已经提供了一个 interval_in() 的动态加载函数，位于 timestamp.c，定义 snippet 如下:

```c
Datum
interval_in(PG_FUNCTION_ARGS)
{
	char	   *str = PG_GETARG_CSTRING(0);

    #ifdef NOT_USED
	Oid			typelem = PG_GETARG_OID(1);
    #endif
	int32		typmod = PG_GETARG_INT32(2);
	Interval   *result;
```

可以看到它接收三个参数：输入的字符串，一个 oid，一个 typmod，但后两个参数却不解其意，这时可以在整个 PG 源码中搜索对 interval_in() 的调用，发现后两个参数似乎也可以不指定，那么这时就可以开始调用了。

因为 interval_in() 接收三个参数，所以使用 DirectFunctionCall3()；

传入的参数全都都要使用 Datum 形式，需要调用各类准备好的宏将你的参数转换为 Datum；

```c
Datum interval = DirectFunctionCall3(interval_in,
							         CStringGetDatum(“1 year 3 months”),
							         ObjectIdGetDatum(InvalidOid),
							         Int32GetDatum(-1));
```

调用完成后得到的 interval 也是 Datum 形式，可以转换为正常的 interval 类型，使用宏：

```c
Interval *interval_p = DatumGetIntervalP(interval);
```

**上面的例子其实是一个错误示例**，因为根据[官方邮件里的](https://www.postgresql.org/message-id/flat/491CCBF5.9020304%40purdue.edu#491CCBF5.9020304@purdue.edu)的说法：

> You should be using InputFunctionCall to invoke any datatype inputfunction.  There are plenty of examples to follow in the standard PLs.

所以对于输入函数，一般需要使用 InputFunctionCall 来调用，比如 array_in() 如果使用 DirectFunctionCall3() 的话，会遇到运行时错误，interval_in() 可能是歪打正着吧，不过**其他的普通动态加载函数都可以如上的示例般使用**。

下面介绍我探索的 InputFunctionCall 调用方法，我觉得有些繁琐，若有更好的方法，欢迎指正。

## 数据类型输入函数

### 使用位于 fmgr.h 中的各类 OidInputFunctionCall 函数

想将一个 '{1,2}' 形式的字符串，转换为 PG 内置的 int[] 类型，使用 array_in() 函数，定义如下：

```c
Datum
array_in(PG_FUNCTION_ARGS)
{
	char	   *string = PG_GETARG_CSTRING(0);	/* external form */
	Oid			element_type = PG_GETARG_OID(1);		/* type of an array
														 * element */
	int32		typmod = PG_GETARG_INT32(2);	/* typmod for array elements */
```

需要传入的参数是：输入的字符串， array 中元素的类型 Oid 以及这个类型的 typmod。

对于元素的类型和 typmod 可以这样获得：

```c
parseTypeString("integer", &typeid_p, &typmod_p, missing_ok);
```

integer 类型的 Oid 和 typmod 就在 typeid_p 和 typmod_p 中。

而 OidInputFunctionCall 函数的调用要求知道函数 Oid，

```c
Datum
OidInputFunctionCall(Oid functionId, char *str, Oid typioparam, int32 typmod);
```

所以通过 fmgr_internal_function 获取其 Oid：

```c
Oid funcoid = fmgr_internal_function("array_in");
```

最后开始调用：

```c
ArrayType *array = DatumGetArrayTypeP(OidInputFunctionCall(funcoid, "{1,2}",
					                         	           typeid_p,
								   			               typmod_p));
```