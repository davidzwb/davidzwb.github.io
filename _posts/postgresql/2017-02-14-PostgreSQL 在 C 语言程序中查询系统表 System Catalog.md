---
layout: post
title:  "PostgreSQL 在 C 语言程序中查询系统表 System Catalog"
date:   2017-02-14 09:40 +0800
categories: postgresql 
---

# PostgreSQL 在 C 语言程序中查询系统表 System Catalog

## 查找某个系统对象的 OID：

> 由于本人也在学习阶段，目前仅能分享在 System Catalog 中查询对象 OID 的方法。

需要关注的函数是 GetSysCacheOid1/GetSysCacheOid2/GetSysCacheOid3/GetSysCacheOid4 系列函数，

通过它们来查询 System Catalog 中对象的 OID。

### 示例：

示例 1，查找一个命名空间 namespace 'public '的 OID：

```c
namespaceId = GetSysCacheOid1(NAMESPACENAME,
							  CStringGetDatum("public"));
```

示例 2，在 namespaceId 这个 OID 对应的命名空间中查找 'integer' 类型的 OID：

```c
GetSysCacheOid2(TYPENAMENSP,
				PointerGetDatum("integer"),
				ObjectIdGetDatum(namespaceId));
```

### 总结：

1. 在 /utils/syscache.h 中的 SysCacheIdentifier 枚举类型中定义了很多查询类别，如 NAMESPACENAME 和 TYPENAMENSP，每个的使用方法可以在整个源码中查找并推断出。
2. GetSysCacheOid1/GetSysCacheOid2/GetSysCacheOid3/GetSysCacheOid4 系列函数后的数字代表查找键的个数，不同的查询需要不同个数的查找键，所以需要调用对应的那个 GetSysCacheOid 函数。
