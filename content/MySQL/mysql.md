+++
date = '2025-09-27T10:27:15+08:00'
draft = false
title = '数据库'
+++

数据库分为关系型数据库和非关系型数据库：

- 关系型数据库是一种基于关系的数据库，数据以表格的形式组织和存储。每个表格分为列和行，列代表一种属性，行代表一条数据。常见的有 MySQL
- 非关系型数据库通常不采用表格结构来存储数据，而是采用各种不同的数据模型来存储，常见的有 Redis 的键值对型

## 数据库三范式

这是用来设计关系型数据库时的三个优化步骤，目的是减少数据冗余、提高数据一致性。

- 第一范式要求数据库的每一列都是不可再分的原子数据项。

  > 例如有个字段为 address 它的值可以是中国北京，那么这个字段就能够被分为 country 和 city 字段

- 第二范式要求必须先满足第一范式，且每一列都完全依赖于主键，不能只依赖于主键的一部分

  > 这主要是针对符合主键的情况。
  >
  > 例如有一个课程表,主键是（学号, 课程）。`分数` 依赖于（学号, 课程）。但 `姓名` 只依赖于 `学号` （部分依赖）。
  >
  > 解决：把 `姓名` 拆出去放到学生表。

- 第三范式要求必须先满足第二范式，且非主键之间不能有传递依赖。即非主键字段只能依赖于主键，而不能依赖于其他非主键字段

  > 例如有一个学生类表，主键是 `学号`。`系名` 依赖 `学号` 。`系主任` 依赖 `系名` （传递依赖）
  >
  > 解决：把 `系名-系主任` 拆出去做一个独立的系表。

## SQL

SQL 是一门操作数据库的脚本语言

### 查询数据

##### 基本查询

```sql
SELECT * FROM `table_name`; # 用来查询全部数据
# SELECT 语句也不一定需要 FROM 子句
```

##### 条件查询

```sql
SELECT * FROM `table_name` WHERE <条件表达式>;
# 不同条件之间可以使用 AND / OR 连接，也可以使用 NOT <条件> 来表示不符合条件
```

##### 投影查询

条件查询`SELECT * FROM table_name`后返回的数据和原表数据是相同的，即结果集的所有列于原有表的所有列一一对应；

如果只返回某些列的数据，我们可以使用

```sql
SELECT 列1, 列2 FROM `table_name`;
```

让结果集只包含指定列，这种操作称为投影查询。

##### 排序

```sql
SELECT column1 FROM `table_name` ORDER BY column2; # ORDER BY 默认按照 ASC 列的大小将查询的数据按从低到高排序
SELECT column1 FROM `table_name` ORDER BY column2 DESC; # DESC 表倒序
```

##### 分页查询

```sql
SELECT * FROM `table_name` LIMIT pageSize OFFSET pageIndex;
# MySQL 中也可以写成 LIMIT offset, row_count
```

##### 聚合查询

sql 提供以下常用聚合函数

| 函数 | 说明                                   |
| ---- | -------------------------------------- |
| SUM  | 计算某一列的合计值，该列必须为数值类型 |
| AVG  | 计算某一列的平均值，该列必须为数值类型 |
| MAX  | 计算某一列的最大值                     |
| MIN  | 计算某一列的最小值                     |

其中`MAX()`和`MIN()`不仅限于数字，如果是字符类型，两个函数分别会返回排序最后和排序最前的字符

使用 sql

```sql
SELECT COUNT(*) FROM `table_name`;
# 表示查询所有列的行数，聚合的结果虽然是一个数字，但查询的结果仍然是一个二维表，只是这个二维表只有一行一列，并且列名是COUNT(*)
|COUNT(*)|
|--------|
| result |
```

聚合函数常与分组一起使用

```sql
SELECT COUNT(*) FROM `table_name` GROUP BY column1;
# 返回结果
|COUNT(*)|
|--------|
|result1 |
|--------|
|result2 |
...
```

这时返回的结果不再是一个，而是根据`column1`的分组数

这时查询结果再加上分组列，就会得到一个更清晰的表

```sql
SELECT column1, COUNT(*) FROM `table_name` GROUP BY column1;
# 返回结果
|column1|COUNT(*)|
|----------------|
|co_val1|result1 |
|----------------|
|co_val2|result2 |
...
```

##### 多表查询

`SELECT` 可以从多张表查询

```sql
SELECT * FROM table_name1, table_name2 ...
# 这样返回的结构会造成笛卡尔积
```

##### 连接查询

连接查询是另一种类型的多表查询。连接查询对多个表进行 JOIN 运算，简单的说，就是先确定一个主表作为结果集，然后把其他表的行有选择性的连接在主表结果集上。

```sql
SELECT t1.column, t2.column
FROM table_name1 t1
INNER JOIN table_name2 t2
ON t1.col_id = t2.col_id; # 内连接，只返回符合 t1.col_id = t2.col_id 的数据可以省略 INNER
SELECT t1.column, t2.column
FROM table_name1 t1
LEFT JOIN table_name2 t2
ON t1.col_id = t2.col_id; # 左外连接，返回连接表的全部数据加符合条件的数据，不符合的位置填 null
SELECT t1.column, t2.column
FROM table_name1 t1
RIGHT JOIN table_name2 t2
ON t1.col_id = t2.col_id; # 右外连接，返回被连接表的全部数据加符合条件的数据，不符合的位置填 null
# MySQL 本身不支持 FULL OUTER JOIN，如果确实需要类似能力，通常要用 LEFT JOIN / RIGHT JOIN 再配合 UNION 实现
```

### 修改数据

##### 插入数据

```sql
INSERT INTO table_name (column1, column2, ...) VALUES (value1, value2, ...)
# 可以一次添加多行数据
```

##### 更新数据

```sql
UPDATE table_name SET column1=value1, column2=value2, ... WHERE ...;
```

##### 删除数据

```sql
DELETE FROM table_name WHERE ...;
```

### 修改表

```sql
ALTER TABLE table_name ADD column_name 数据类型 约束; # 增加新列
ALTER TABLE table_name MODIFY COLUMN column_name 新数据类型; # MySQL 中通常使用 MODIFY COLUMN 改变列的数据类型
ALTER TABLE table_name DROP COLUMN column_name; # 删除列
ALTER TABLE table_name RENAME TO new_table_name; # 修改表名
```

## MySQL

### 存储引擎

MySQL 有很多存储引擎。它是用来存储、检索、更新和删除数据的服务。

MySQL 中可以为每个表指定存储引擎

```sql
CREATE TABLE t (
    id INT
) ENGINE = INNODB;
```

可以使用`SHOW ENGINES`查看数据库中的所有存储引擎。

我们最常用的存储引擎有三种：

- InnoDB：是 MySQL 5.5 之后的默认存储引擎，它支持事务、外键、崩溃恢复和行级锁。它更偏向通用 OLTP 场景，综合能力强，但相对会占用更多存储空间。
- MyISAM：是 MySQL 5.5 之前的默认存储引擎，不支持事务、行级锁和外键等特性。它主要依赖表级锁，更适合读多写少且对事务要求不高的场景。
- MEMORY：内存型存储引擎，所有的数据都存储在内存中，因此它的读写效率很高，但在 MySQL 服务重启后会丢失数据，并且不支持事务、外键。

### 索引

#### 索引类型

索引一般分为：

- 主键索引：用于唯一标识每一行数据的索引，每个表只能有一个主键索引

  > 主键索引需要在创建表时指定：
  >
  > ```sql
  > CREATE TABLE table_name (
  > 	column1 INT PRIMARY KEY,
  >     ...
  > )
  > ```

- 唯一索引：用于保证列的唯一性的索引，一个表可以有多个唯一索引

  > 创建唯一索引：
  >
  > ```sql
  > CREATE UNIQUE INDEX index_name
  > ON table_name (column)
  > ```

- 普通索引：普通索引也叫非唯一索引，它是最常见的一种索引类型，可以加速查询和排序操作

  > 创建普通索引：
  >
  > ```sql
  > CREATE INDEX index_name
  > ON table_name (column);
  > ```

- 全文索引：文索引是一种用于全文搜索的索引类型，能够对文本数据进行快速的模糊搜索和关键字搜索

  > 创建全文索引：
  >
  > ```sql
  > CREATE FULLTEXT INDEX index_name
  > ON table_name (column);
  > ```

- 复合索引：也叫多列索引或联合索引，它是包含多个列的索引类型，能够加速多列查询和排序操作

  > 创建复合索引：
  >
  > ```sql
  > CREATE INDEX index_name
  > ON table_name (column1, column2, ...);
  > ```

- 哈希索引：基于哈希表实现的索引类型，能够对等值查询进行高效的处理，但不支持范围查询和排序，MySQL 中 Memory 引擎中支持哈希索引

> ```sql
> # 查看索引
> SHOW INDEX FROM table_name;
> SHOW CREATE TABLE table_name;
> # 删除索引
> DROP INDEX index_name ON table_name;
> ```

来看一下**外键约束 (FOREIGN KEY)** 

外键用于建立两个表之间的练习

使用索引的优点：

- 当没有索引时，数据库会进行全表扫描，耗时。使用索引可以加速数据的检索效率，提高查询效率

  > 因此如下情况可以考虑创建索引
  >
  > - 频繁用于查询的列
  > - 经常用于排序的列
  > - 经常用于聚合函数的列

- 在列上创建唯一索引和主键索引可以确保数据的唯一性和完整性

缺点：

- 创建索引会占用额外的储存空间

- 改变数据的同时需要额外的维护索引

  > 因此对于读多写少的表，索引会增加写操作的时间，并占用更多的存储空间，不宜过多地建立索引

- 有些情况下使用索引可能会导致查询效率降低甚至出现索引失效的情况。

  > 当对于一个非常小的表或者一个稠密的索引列进行查询时，使用索引可能并不会提高查询效率
  >
  > 索引失效的场景：
  >
  > - 在使用复合索引时，只有从索引的最左边的列开始查询才能用到该符合索引
  > - 如果对索引列使用运算，索引会失效。如 `WHERE id + 1 = 2`
  > - 查询列如果使用任意 MySQL 提供的函数会导致索引失效
  > - 如果索引列存在类型转换会导致索引失效
  > - 当在查询中使用了 is not null 也会导致索引失效，而 is null 则会正常触发索引的
  > - 只有 like '()%' 这种形式的模糊查询索引不会失效

#### 索引的底层数据结构

MySQL 中默认存储引擎 InnoDB 的索引底层主要使用 **B+ 树**。B+ 树是一种多路搜索树，非叶子节点主要保存索引键值，叶子节点保存索引记录，并且叶子节点之间通常按顺序相连，因此非常适合范围查询和排序。

```text
B+ 树结构：

             [10 | 20]
           /     |     \
       [1,5,8] [12,15,18] [22,25,28]
# [10,20] 的含义是：根节点把数据划分成 <10、[10,20)、>=20 三段范围。
- 根节点 [10,20]：快速定位
- 内部节点只存键，不存行数据
- 叶子节点 [1,5,8]、[12,15,18]、[22,25,28] 存实际数据
- 叶子节点通过指针左右相连，方便范围查询
```

与普通 B 树相比，B+ 树的非叶子节点通常不保存完整行记录，因此单个节点可以容纳更多键值，整棵树的高度往往更低，磁盘 IO 次数也更可控。

B+ 树的叶子节点按顺序链接，所以它在范围查询、排序扫描这类场景里表现尤其好，这也是它适合数据库索引的重要原因。

#### 索引在存储结构上的组织方式

**聚簇索引**：

也被称为聚集索引，在 InnoDB 存储引擎中，每个表只能有一个聚集索引，其余的索引都是非聚集索引（也称为二级索引）。聚集索引是按照数据在磁盘上的物理顺序来组织数据的，其叶子节点保存着完整的数据行信息。InnoDB 中，如果表定义了主键，则主键索引是聚集索引；如果表没有定义主键，则第一个唯一非空索引是聚集索引；如果都没有，则 InnoDB 会隐式创建一个隐藏的聚集索引。

**非聚簇索引**：

非聚集索引也叫二级索引，其叶子节点保存着索引字段和指向对应数据行的指针（相当于主键 ID），通过这个指针可以找到对应的数据行。在查询中，如果使用的是非聚集索引，则需要先根据索引查找到对应的行指针，再通过行指针查找数据行，这个过程叫做回表查询。

> 由此可以看出，聚簇索引占用更大的存储空间。而在查询数据时，由于非聚簇索引需要进行回表操作，所以相对于聚簇索引要慢一些
