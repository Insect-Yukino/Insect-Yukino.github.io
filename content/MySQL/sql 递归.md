+++
date = '2026-02-19T17:35:55+08:00'
draft = false
title = 'sql 递归'
+++

递归语法：

```sql
WITH RECURSIVE cte AS (
  -- 1) anchor：递归起点（第一层）
  SELECT ...
  FROM ...
  WHERE ...

  UNION ALL

  -- 2) recursive：递归体（用上一次的结果继续查下一层）
  SELECT ...
  FROM ...
  JOIN cte ON ...
)
SELECT * FROM cte;

```

**最后一行** 只是结果集按某种排序后的最后一个节点，它并不天然等于一棵递归树。

下面我用最常见的部门表举例：`dept(id, parent_id, name)`。

#### 1) 递归后的结果集：每一行就是一个节点

SQL：

```sql
WITH RECURSIVE dept_tree AS (
  SELECT id, parent_id, name, 0 AS level
  FROM dept
  WHERE id = 100          -- 起点

  UNION ALL

  SELECT d.id, d.parent_id, d.name, t.level + 1
  FROM dept d
  JOIN dept_tree t ON d.parent_id = t.id
)
SELECT id, parent_id, name, level
FROM dept_tree
ORDER BY level, id;
```

假设数据是这样一棵树：

- 100 总部
  - 110 研发
    - 111 后端
    - 112 前端
  - 120 财务

递归结果集（扁平）可能是：

| id   | parent_id | name | level |
| ---- | --------- | ---- | ----- |
| 100  | NULL      | 总部 | 0     |
| 110  | 100       | 研发 | 1     |
| 120  | 100       | 财务 | 1     |
| 111  | 110       | 后端 | 2     |
| 112  | 110       | 前端 | 2     |

那么递归后的一行数据举例就是：

```sql
id=111, parent_id=110, name='后端', level=2
```
