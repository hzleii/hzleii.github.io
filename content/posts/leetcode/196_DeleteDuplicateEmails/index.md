---
title: "196.删除重复的电子邮箱"
description: "196.删除重复的电子邮箱"
date: 2020-04-08
lastmod: 2020-04-08
math:
  enable: true
categories: ["LeetCode"]
tags: ["LeetCode"]
---


## 题目描述

编写一个 SQL 查询，来删除 Person 表中所有重复的电子邮箱，重复的邮箱里只保留 Id 最小 的那个。

**示例：**

```
+----+------------------+
| Id | Email            |
+----+------------------+
| 1  | john@example.com |
| 2  | bob@example.com  |
| 3  | john@example.com |
+----+------------------+
Id 是这个表的主键。
```

**在运行你的SQL语句之后，上面的 Person 表应返回以下几行：**

```
+----+------------------+
| Id | Email            |
+----+------------------+
| 1  | john@example.com |
| 2  | bob@example.com  |
+----+------------------+
```

**说明：**

- 执行 SQL 之后，输出是整个 Person 表。
- 使用 delete 语句。



## 题解

## 方法：使用 DELETE 和 WHERE 子句

**思路：**

我们可以使用以下代码，将此表与它自身在电子邮箱列中连接起来。

```sql
SELECT p1.*
FROM Person p1,
    Person p2
WHERE
    p1.Email = p2.Email
;
```

然后我们需要找到其他记录中具有相同电子邮件地址的更大 ID。所以我们可以像这样给 WHERE 子句添加一个新的条件。

```sql
SELECT p1.*
FROM Person p1,
    Person p2
WHERE
    p1.Email = p2.Email AND p1.Id > p2.Id
;
```

因为我们已经得到了要删除的记录，所以我们最终可以将该语句更改为 **DELETE**。

```sql
DELETE p1 FROM Person p1,
    Person p2
WHERE
    p1.Email = p2.Email AND p1.Id > p2.Id
```

