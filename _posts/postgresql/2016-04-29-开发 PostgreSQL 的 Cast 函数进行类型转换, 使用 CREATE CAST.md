---
layout: post
title:  "开发 PostgreSQL 的 Cast 函数进行类型转换, 使用 CREATE CAST"
date:   2016-04-29 11:43 +0800
categories: postgresql
---

# 开发 PostgreSQL 的 Cast 函数进行类型转换, 使用 CREATE CAST 

```
CREATE CAST (source_type AS target_type)
WITH FUNCTION function_name (argument_type [, ...])
[ AS ASSIGNMENT | AS IMPLICIT ]

CREATE CAST (source_type AS target_type)
WITHOUT FUNCTION
[ AS ASSIGNMENT | AS IMPLICIT ]

CREATE CAST (source_type AS target_type)
WITH INOUT
[ AS ASSIGNMENT | AS IMPLICIT ]
```

## 1. 创建方法:

先使用 CREATE FUNCTION function_name (argument_type [, ...]) 创建一个转换函数;

然后 CREATE CAST (source_type AS target_type)
WITH FUNCTION function_name (argument_type [, ...])

## 2. 创建时可选择的选项的说明:

一般转换时的语句如下:

CAST(x AS typename) or x::typename

当两种类型的底层表示相同时, 可以使用 WITHOUT FUNCTION;
若想使用现成的 source_type 的 out 函数和 target_type 的 in 函数, 则可以使用 WITH INOUT;
当指定 AS ASSIGNMENT 时, 则当对表的一列进行赋值, 如 INSERT INTO 操作时, 会自动隐式地调用转换函数;
当指定 AS IMPLICIT 时, 则对于任何场合, 都会自动隐式地调用转换函数, 这个需要谨慎使用, 文档上有说明. 