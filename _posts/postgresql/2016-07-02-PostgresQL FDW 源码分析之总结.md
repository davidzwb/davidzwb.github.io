---
layout: post
title:  "PostgreSQL FDW 源码分析之总结"
date:   2016-07-02 22:35 +0800
categories: postgresql 
---

函数的阅读顺序如下：

1. FDW 先在 GetForeignRelSize() 中通过本地或向 remote 端查询的方式，得到了对 SQL 语句基本扫描方式的成本估算；
2. 然后在 postgresGetForeignPaths() 计算出各种情况的 plan path，包括基础扫描方法，和各种 join 方法，path 中带有对这种 plan 处理方式的描述和成本估算；
3. 在从多个 plan path 中选出一个 best path 后，在 postgresGetForeignPlan() 中，从这个 path 中恢复出需要在执行器执行的 SQL 语句；
4. 在 postgresBeginForeignScan() 中作一些资源申请等准备工作；
5. 最后在 postgresIterateForeignScan() 中，在 remote 建立要执行 SQL 语句的游标，然后通过游标取回查询结果。

要点：

1. FDW 会区分 where 或 join 等限制条件，将能够发送到 remote 端的限制条件发送到 remote 端执行，此时就只需从 remote 端仅取回需要的数据了。
2. 截止到 9.6，已经支持 join 条件的 remote 端下推了，在 10 还将支持聚合函数的下推。