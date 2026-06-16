+++
date = '2026-02-17T23:06:17+08:00'
draft = false
title = 'Schema 模式'
+++

我第一次在实习公司使用 PostgreSQL 数据库，第一次接触到了 **Schema（模式）设计**。

在 **PostgreSQL** 里，命名空间 = **schema**，它是数据库内部的逻辑分组容器。

## 一、什么是 PostgreSQL 的命名空间？

结构层级是这样的：

```text
数据库（Database）
    └── 模式（Schema）← 命名空间
            ├── 表（Table）
            ├── 视图（View）
            ├── 函数（Function）
            ├── 类型（Type）
            ├── 序列（Sequence）
```

比如：

```text
public.user
auth.user
```

这里：

- `public` 是 schema
- `auth` 是 schema
- `user` 是表

这两个表可以同时存在，因为它们属于不同命名空间。

## 二、为什么要设计命名空间？

### 解决重名问题

不同 schema 可以有同名对象：

```text
public.order
test.order
```

避免表名冲突。

### 模块化组织数据库

例如：

```text
auth.users
auth.roles
order.orders
order.order_items
```

像包结构一样管理数据库对象。

### 权限隔离

可以给 schema 授权：

```sql
GRANT USAGE ON SCHEMA auth TO app_user;
```

比单表授权更方便。

### 多租户支持

不同租户使用不同 schema：

```text
tenant_a.user
tenant_b.user
```

逻辑隔离，物理仍在同一数据库。

## 三、默认命名空间机制

PostgreSQL 有一个默认 schema：

```text
public
```

当你执行：

```sql
SELECT * FROM user;
```

实际上 PostgreSQL 会根据 `search_path` 去找：

```sql
SHOW search_path;
```

默认通常是：

```sql
"$user", public
```

意思是：

1. 先找与你用户名同名的 schema
2. 再找 public

## 四、search_path 是关键

`search_path` 决定解析顺序。

如果：

```text
SET search_path TO auth, public;
```

执行：

```sql
SELECT * FROM user;
```

优先找：

```
auth.user
```

找不到再去 public。

## 五、和 MySQL 的区别

在 **MySQL** 里：

```text
数据库 = 命名空间
```

没有 schema 概念。

而 PostgreSQL 是：

```text
数据库
   └── 多个 schema（命名空间）
```

PostgreSQL 更细粒度。

## 六、底层实现原理

在 PostgreSQL 内部：

- 每个 schema 在系统表 `pg_namespace` 里
- 每个对象（表、函数等）都有一个 namespace OID
- 通过 namespace + name 唯一确定对象

我实习公司里面的项目，有一个我感觉非常反人类的设计，他将不同的 schema 当作不同的数据源，通过 MP 多数据源为每一个 schema 分一个 JDBC 连接。要知道，这个项目并没有分布式事务。当一个业务操作横跨多个 schema 时，不同的 JDBC 连接不可以被认做一个事务，在一些操作中，就不能用事务保证操作的完整性。因此不得不放弃事务，因为该项目没有这种情况的解决方案，我一个实习生人微言轻。
