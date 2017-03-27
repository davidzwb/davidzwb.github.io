---
layout: post
title:  "PostgreSQL 数据库统计信息查看"
date:   2016-05-12 14:27 +0800
categories: postgresql 
---

Table 9-73. Database Object Size Functions

| Name                                     | Return Type | Description                              |
| ---------------------------------------- | ----------- | ---------------------------------------- |
| `pg_column_size(any)`                    | `int`       | Number of bytes used to store a particular value (possibly compressed) |
| `pg_database_size(oid)`                  | `bigint`    | Disk space used by the database with the specified OID |
| `pg_database_size(name)`                 | `bigint`    | Disk space used by the database with the specified name |
| `pg_indexes_size(regclass)`              | `bigint`    | Total disk space used by indexes attached to the specified table |
| `pg_relation_size(relation regclass, fork text)` | `bigint`    | Disk space used by the specified fork (`'main'`, `'fsm'`, `'vm'`, or `'init'`) of the specified table or index |
| `pg_relation_size(relation regclass)`    | `bigint`    | Shorthand for `pg_relation_size(..., 'main')` |
| `pg_size_pretty(bigint)`                 | `text`      | Converts a size in bytes expressed as a 64-bit integer into a human-readable format with size units |
| `pg_size_pretty(numeric)`                | `text`      | Converts a size in bytes expressed as a numeric value into a human-readable format with size units |
| `pg_table_size(regclass)`                | `bigint`    | Disk space used by the specified table, excluding indexes (but including TOAST, free space map, and visibility map) |
| `pg_tablespace_size(oid)`                | `bigint`    | Disk space used by the tablespace with the specified OID |
| `pg_tablespace_size(name)`               | `bigint`    | Disk space used by the tablespace with the specified name |
| `pg_total_relation_size(regclass)`       | `bigint`    | Total disk space used by the specified table, including all indexes and TOAST data |

所以，比如要查看 test 数据库的大小，则输入：

```
select pg_size_pretty(pg_database_size('test'));
```