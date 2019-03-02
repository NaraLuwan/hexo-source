---
title: MySQL 中的 null 值处理
tags: [MySQL]
categories:
  - MySQL
date: 2019-01-09 19:53:48
---

## 问题

今天在项目中遇到一个问题，查询数据库（MySQL）一张表数据时一直返回空列表，但数据库里是有数据的，排查后发现是由于 where 条件里有一个 not in 筛选条件，但是表数据里那个字段存在 null 值。

模拟一下场景，从 user 表中查询 user_name 不在指定列表中，只要列表中出现一个 null 值，返回总是空列表。sql 如下：

```sql
SELECT * FROM `user` WHERE `user_name` NOT IN ('xx', NULL ,'xx');
```

由于在 MySQL 中 null 代表的是一个未知的值，因而 not in 条件在筛选时所有数据都不符合，包括 user_name 字段为 null 的数据。

## 思考

1. not null 和 null 的区别？分别在什么情况下使用？

<!-- more -->

> NULL means you do not have to provide a value for the field.
  NOT NULL means you must provide a value for the fields.
  For example, if you are building a table of registered users for a system, you might want to make sure the user-id is always populated with a value (i.e. NOT NULL), but the optional spouses name field, can be left empty (NULL)

引用 stackoverflow 上的一个回答。not null 意味着每条数据这个字段必须有一个确定的值，而 null 表示可以有也可以没有。

查了下 MySQL 官方文档，对 null 值是这样描述的：

> NULL columns require additional space in the row to record whether their values are NULL. For MyISAM tables, each NULL column takes one bit extra, rounded up to the nearest byte.

Null 列需要更多的存储空间。所以存储空间上 not null 优于 null。

2. 允许为 null 时应该注意哪些细节？

- 就一条原则：给允许为 null 的字段设置默认值，避免出现于 null 值比较。

<hr />