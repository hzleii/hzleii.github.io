---
title: "184.部门工资最高的员工"
description: "184.部门工资最高的员工"
date: 2020-04-05
lastmod: 2020-04-05
math:
  enable: true
categories: ["LeetCode"]
tags: ["LeetCode"]
---



## 题目描述

**Employee** 表包含所有员工信息，每个员工有其对应的 **Id**, **salary** 和 **department Id**。

**示例：**

```
+----+-------+--------+--------------+
| Id | Name  | Salary | DepartmentId |
+----+-------+--------+--------------+
| 1  | Joe   | 70000  | 1            |
| 2  | Jim   | 90000  | 1            |
| 3  | Henry | 80000  | 2            |
| 4  | Sam   | 60000  | 2            |
| 5  | Max   | 90000  | 1            |
+----+-------+--------+--------------+
```

**Department** 表包含公司所有部门的信息。

```
+----+----------+
| Id | Name     |
+----+----------+
| 1  | IT       |
| 2  | Sales    |
+----+----------+
```

编写一个 SQL 查询，找出每个部门工资最高的员工。对于上述表，您的 SQL 查询应返回以下行（行的顺序无关紧要）。

```
+------------+----------+--------+
| Department | Employee | Salary |
+------------+----------+--------+
| IT         | Max      | 90000  |
| IT         | Jim      | 90000  |
| Sales      | Henry    | 80000  |
+------------+----------+--------+
```

**说明：**

- Max 和 Jim 在 IT 部门的工资都是最高的，Henry 在销售部的工资最高。



## 题解

## 方法：使用 JOIN 和 IN 子句

**思路：**

因为 **Employee** 表包含 **Salary** 和 **DepartmentId** 字段，我们可以以此在部门内查询最高工资。

```sql
SELECT
    DepartmentId, MAX(Salary)
FROM
    Employee
GROUP BY DepartmentId;
```

<p style="color: red; font-weight: bold;">注意：有可能有多个员工同时拥有最高工资，所以最好在这个查询中不包含雇员名字的信息。</p>

```
+--------------+-------------+
| DepartmentId | MAX(Salary) |
|--------------|-------------|
| 1            | 90000       |
| 2            | 80000       |
+--------------+-------------+
```

然后，我们可以把表 **Employee** 和 **Department** 连接，再在这张临时表里用 **IN** 语句查询部门名字和工资的关系。

```sql
SELECT
    Department.name AS 'Department',
    Employee.name AS 'Employee',
    Salary
FROM
    Employee
        JOIN
    Department ON Employee.DepartmentId = Department.Id
WHERE
    (Employee.DepartmentId , Salary) IN
    (   SELECT
            DepartmentId, MAX(Salary)
        FROM
            Employee
        GROUP BY DepartmentId
	)
;
```

结果

```
+------------+----------+--------+
| Department | Employee | Salary |
|------------|----------|--------|
| Sales      | Henry    | 80000  |
| IT         | Max      | 90000  |
+------------+----------+--------+
```

