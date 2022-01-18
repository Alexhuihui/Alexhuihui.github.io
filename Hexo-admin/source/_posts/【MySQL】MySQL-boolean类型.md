---
title: 【MySQL】MySQL boolean类型
date: 2022-01-17 19:12:49
categories: MySQL
tags: MySQL
---

### MySQL BOOLEAN数据类型

MySQL没有内置的布尔类型。 但是它使用`TINYINT(1)`。 为了更方便，MySQL提供`BOOLEAN`或`BOOL`作为`TINYINT(1)`的同义词。在MySQL中，`0`被认为是`false`，非零值被认为是`true`。

<!-- more --> 

#### MySQL BOOLEAN示例

MySQL将布尔值作为整数存储在表中，我们建立一张`tasks`表来作为示例展示。

```mysql
CREATE TABLE tasks (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255) NOT NULL,
    completed BOOLEAN
);
```

我们在建表语句中将 `completed`设为`boolean`类型，但是展示表结构的时候它会变为 `tinyint(1)`。

```mysql
CREATE TABLE `tasks` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `title` varchar(255) NOT NULL,
  `completed` tinyint(1) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8mb4;
```

我们向表中插入两条数据

```mysql
INSERT INTO `Test`.`tasks` (`id`, `title`, `completed`) VALUES (2, '测试', true);
INSERT INTO `Test`.`tasks` (`id`, `title`, `completed`) VALUES (3, '测试2', false);
```

MySQL会在存储数据之前，把`true`和`false`转换成`1`和`2`再保存，下图是查询结果

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/20220118112424.png)

因为`Boolean`类型是`tinyint(1)`的同义词，所以可以在布尔列中插入`1`和`0`以外的值。如下示例：

```mysql
INSERT INTO `Test`.`tasks` (`id`, `title`, `completed`) VALUES (1, '打算', NULL);
INSERT INTO `Test`.`tasks` (`id`, `title`, `completed`) VALUES (4, 'MySQL', 2);
```

现在表中的数据如下:

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/20220118112750.png)

如果要将结果输出为`true`和`false`，可以使用`IF`函数，如下所示：

```mysql
SELECT 
    id, 
    title, 
    IF(completed, 'true', 'false') completed
FROM
    tasks;
```

执行以上语句，结果如下

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/20220118112953.png)

---

#### MySQL BOOLEAN运算符

如果你想查询表中所有 `completed`为`true`的数据，那么就应该使用`IS`运算符而不是`=`

```mysql
SELECT 
    id, title, completed
FROM
    tasks
WHERE
    completed = TRUE
```

执行以上语句，结果如下:

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/20220118113401.png)

它只返回`completed`列值为1的数据，要想返回所有为`true`的则必须使用 `IS`运算符

```mysql
SELECT 
    id, title, completed
FROM
    tasks
WHERE
    completed IS TRUE;
```

执行结果:

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/20220118113611.png)

当然你还可以使用 `IS FALSE` 或者`IS NOT TRUE`，来查询所有待处理的数据。必须注意的是，前者只能查询出`completed`等于`0`的数据，后者还可以查询出`completed`等于`null`的数据。

![](https://raw.githubusercontent.com/Alexhuihui/photo/main/20220118113929.png)

