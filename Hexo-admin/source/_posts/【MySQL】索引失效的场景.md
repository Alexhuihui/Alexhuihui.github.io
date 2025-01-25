---
title: 【MySQL】索引失效的场景
date: 2024-05-22 19:54:06
categories: MySQL
tags: MySQL
---

今天我们讨论一下数据库索引在什么情况下会失效，总的来说有以下几种场景

1. 在索引列上加函数运算
2. 组合索引中，不符合最左匹配原则
3. 当索引列存在隐式转化的时候
4. 使用like通配符匹配后缀%xxx的时候
5. 使用or连接查询的时候，or语句前后没有同时使用索引

<!-- more --> 

### 设计 SQL 表

假设我们设计一个员工表 `employees`，包含以下字段：

- `employee_id`：员工ID（字符串类型，作为索引列）
- `name`：员工姓名
- `department_id`：部门ID（字符串类型，作为索引列）
- `age`：员工年龄

### 创建表的 SQL 语句

```
CREATE TABLE employees (
    employee_id VARCHAR(20) NOT NULL,
    name VARCHAR(100),
    department_id VARCHAR(20),
    age INT,
    PRIMARY KEY (employee_id),
    KEY idx_department_id (department_id)
);
```

### 插入示例数据

```
INSERT INTO employees (employee_id, name, department_id, age) VALUES
('E001', 'Alice', 'D001', 30),
('E002', 'Bob', 'D002', 25),
('E003', 'Charlie', 'D001', 28),
('E004', 'David', 'D003', 35),
('E005', 'Eve', 'D002', 22);
```

### 查询语句及其 EXPLAIN 分析

#### 1. 隐式类型转换导致索引失效

查询员工ID为123的员工信息：

```
SELECT * FROM employees WHERE employee_id = 123;
```

分析：

```
EXPLAIN SELECT * FROM employees WHERE employee_id = 123;
```

#### 2. LIKE 通配符匹配后缀导致索引失效

查询名字以 ‘lice’ 结尾的员工：

```
SELECT * FROM employees WHERE name LIKE '%lice';
```

分析：

```
EXPLAIN SELECT * FROM employees WHERE name LIKE '%lice';
```

#### 3. OR 连接查询索引失效

查询员工ID为’E001’或部门ID为’D003’的员工信息：

```
SELECT * FROM employees WHERE employee_id = 'E001' OR department_id = 'D003';
```

分析：

```
EXPLAIN SELECT * FROM employees WHERE employee_id = 'E001' OR department_id = 'D003';
```

#### 4. 正确使用索引的查询

查询名字以 ‘A’ 开头的员工：

```
SELECT * FROM employees WHERE name LIKE 'A%';
```

分析：

```
EXPLAIN SELECT * FROM employees WHERE name LIKE 'A%';
```

### EXPLAIN 结果分析

通过 `EXPLAIN` 语句，可以分析每个查询的执行计划。以下是可能的解释：

#### 1. 隐式类型转换导致索引失效

```
+----+-------------+-----------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table     | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-----------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | employees | ALL  | PRIMARY       | NULL | NULL    | NULL |    5 | Using where |
+----+-------------+-----------+------+---------------+------+---------+------+------+-------------+
```

解释：由于隐式类型转换，MySQL 进行了全表扫描（`type=ALL`），未能使用索引。

#### 2. LIKE 通配符匹配后缀导致索引失效

```
+----+-------------+-----------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table     | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-----------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL |    5 | Using where |
+----+-------------+-----------+------+---------------+------+---------+------+------+-------------+
```

解释：由于通配符在前，MySQL 进行了全表扫描（`type=ALL`），未能使用索引。

#### 3. OR 连接查询索引失效

```
+----+-------------+-----------+------+--------------------+------+---------+------+------+-------------+
| id | select_type | table     | type | possible_keys      | key  | key_len | ref  | rows | Extra       |
+----+-------------+-----------+------+--------------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | employees | ALL  | PRIMARY,idx_department_id | NULL | NULL    | NULL |    5 | Using where |
+----+-------------+-----------+------+--------------------+------+---------+------+------+-------------+
```

解释：由于 `OR` 语句使得查询需要考虑多个条件，MySQL 进行了全表扫描（`type=ALL`），未能使用索引。

#### 4. 正确使用索引的查询

```
+----+-------------+-----------+-------+---------------+---------------+---------+------+------+-------------+
| id | select_type | table     | type  | possible_keys | key           | key_len | ref  | rows | Extra       |
+----+-------------+-----------+-------+---------------+---------------+---------+------+------+-------------+
|  1 | SIMPLE      | employees | range | NULL          | idx_name      | 1024    | NULL |    2 | Using where |
+----+-------------+-----------+-------+---------------+---------------+---------+------+------+-------------+
```

解释：由于通配符在后，符合最左匹配原则，MySQL 使用了索引（`type=range`）。

通过以上查询和分析，可以更好地理解 B+树索引在不同情况下的使用和失效原因，从而优化数据库查询性能。