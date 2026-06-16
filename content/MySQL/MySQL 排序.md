+++
date = '2026-03-18T14:55:58+08:00'
draft = false
title = 'MySQL 排序'
+++

MySQL 存在存在多种排序方式 `ORDER BY`

如果 `ORDER BY` 排序字段是索引，那性能是相当的好了。

如果不是索引，那就有可能走 `filesort` 内存排序，读出需要排序的数据，在内存中使用快排完成排序。

如果这部分数据相当的大，MySQL 分配的这块排序内存不足以放下这么多数据，就会使用磁盘临时文件，采用归并排序

对 `ORDER BY` 来说，核心不是“很多种排序算法名字”，而是三条执行路径——按索引顺序直接取、`filesort` 内存排序、`filesort`/临时表落盘。接下来我把它整理成你做业务时最有用的判断顺序。

Thought for 1m 20s

可以把这个问题拆成两层看：

**第一层：MySQL 真正常见的“排序执行方式”并不多。**
 官方文档里，`ORDER BY` 的核心路径其实主要就是两类：**直接按索引顺序取数据**，或者 **做 `filesort`**。另外，某些查询还会伴随**内部临时表**，比如 `ORDER BY` 和 `GROUP BY` 不一致、`DISTINCT + ORDER BY`、或者排序列来自连接中的非首个驱动表时。`EXPLAIN` 里常见的信号就是 `Using filesort` 和 `Using temporary`。

**第二层：优化器不是“任选一种”，而是按代价模型和统计信息选它认为更便宜的方案。**
 MySQL 的代价模型会把 `filesort`、读索引块、读数据块等成本算进去；统计信息和索引基数也会影响它到底选索引顺序扫描、范围扫描还是扫描后再排序。`ANALYZE TABLE` 会更新这些统计信息，统计不准时优化器可能选错计划。

------

## 常见排序方式

### 1）按索引顺序直接返回

这是最理想的情况。
 如果 `ORDER BY` 能被某个索引满足，MySQL 可以**避免额外排序阶段**。多列索引遵循**最左前缀**规则；并且 `ORDER BY` 的列顺序、连续性、方向组合都要和索引匹配。MySQL 8.x 也支持降序索引，混合 `ASC/DESC` 的场景可以通过合适的升降序索引来适配。

典型能走索引排序的例子：

```
SELECT * FROM t ORDER BY a, b;

SELECT * FROM t WHERE a = ? ORDER BY b;

SELECT * FROM t ORDER BY a DESC, b DESC;
```

这类计划下，`EXPLAIN` 通常**不会出现** `Using filesort`。

------

### 2）`filesort`，但能基本在内存里完成

如果索引不能满足 `ORDER BY`，MySQL 会进入 `filesort`。
 注意：`Using filesort` 不等于“一定落磁盘”，它表示**有额外排序阶段**。排序缓冲按需增量分配到 `sort_buffer_size`，结果集较小时可以纯内存完成；某些 `single table + ORDER BY 非索引列 + LIMIT N` 的场景还比较适合纯内存 `filesort`。

------

### 3）`filesort` + 临时文件 / 临时表落盘

这是更重的一类。
 当排序结果放不进内存，`filesort` 会用临时磁盘文件；而且有些 SQL 结构本身就容易触发内部临时表，比如 `ORDER BY` 和 `GROUP BY` 不同、`DISTINCT + ORDER BY`、窗口函数、以及某些复杂连接。内部临时表先在内存里用 `TempTable`/`MEMORY`，超阈值后会转成磁盘上的 `InnoDB` 临时表。

------

## 性能高低怎么排

一般可以这样排：

**覆盖索引顺序返回**
 **> 非覆盖索引顺序返回（还要回表）**
 **> 内存 `filesort`**
 **> 磁盘 `filesort` / 临时表 + 排序**

原因很直接：
 能用索引顺序返回时，MySQL可以省掉额外排序；如果还是覆盖索引，连回表都省了。`filesort` 至少多一轮排序；一旦出现临时文件、merge pass、磁盘临时表，I/O 和 CPU 开销都会明显上去。官方文档也明确说了，代价模型里 `filesort` 会被视为更贵，而 `Using index` 表示可以只靠索引拿列值。

不过这不是绝对定律。
 比如“索引顺序扫描 + 大量随机回表”有时未必比“小结果集内存排序”更快，所以优化器会按成本估算来选。

------

## 哪些写法最容易让排序退化

### 1）`ORDER BY` 列和索引对不上

比如：

```
SELECT * FROM t ORDER BY key1, key2;
```

但你只有 `idx(key1)` 和 `idx(key2)` 两个单列索引，没有一个联合索引 `(key1, key2)`，那通常不能直接靠索引完成排序。官方文档明确举了“`ORDER BY` 用到不同索引”“用到索引的非连续列”这类反例。

### 2）排序列上做表达式

比如：

```
SELECT * FROM t ORDER BY ABS(score);
SELECT * FROM t ORDER BY -create_time;
```

这类写法通常不能直接利用原列索引来排序。

### 3）前缀索引拿来做完整排序

如果只给长字符串建了前缀索引，例如只索引前 10 个字节，那它不能完整区分后续内容，排序仍可能需要 `filesort`。

### 4）多表 join 后按不合适的表字段排序

在 join 里，如果 `ORDER BY` 的列不都来自执行计划里的**第一个非 const 表**，就很容易退化成 `Using temporary` / `Using filesort`。

------

## 代码里怎么优化

### 1）把 `WHERE + ORDER BY` 设计成一个联合索引

这是最重要的。

例如业务 SQL：

```
SELECT id, title, create_time
FROM post
WHERE status = 1
ORDER BY create_time DESC, id DESC
LIMIT 20;
```

更适合的索引是：

```
CREATE INDEX idx_post_status_ctime_id
ON post(status, create_time DESC, id DESC);
```

这样 `status = 1` 用于过滤，后面的 `create_time, id` 用于有序扫描。
 多列索引的最左前缀和降序索引能力，都是官方明确支持的。

------

### 2）给排序补一个稳定的 tiebreaker

比如不要只写：

```
ORDER BY create_time DESC
```

而要写成：

```
ORDER BY create_time DESC, id DESC
```

原因是：当多个行的 `create_time` 相同，MySQL 对这些并列行的返回顺序可以是不确定的，尤其 `LIMIT` 存在时更明显。官方文档明确说了，同一 `ORDER BY` 值的多行在非排序列上的顺序是 nondeterministic。

------

### 3）深分页别用大 `OFFSET`

差的写法：

```
SELECT id, title, create_time
FROM post
WHERE status = 1
ORDER BY create_time DESC, id DESC
LIMIT 100000, 20;
```

这会让 MySQL 先找到并处理前面大量行，再丢掉。
 更好的做法是**seek pagination / keyset pagination**：

```
SELECT id, title, create_time
FROM post
WHERE status = 1
  AND (create_time < :last_create_time
       OR (create_time = :last_create_time AND id < :last_id))
ORDER BY create_time DESC, id DESC
LIMIT 20;
```

这个写法不是 MySQL 文档里的专门术语，但从执行机理上看，它更容易继续沿着索引向后扫描，而不是“跳过 10 万行”。

------

### 4）先拿 ID，再回表取大字段

当行很宽、正文很大时，直接排序全字段很吃亏。可以拆成两步：

```
SELECT id
FROM post
WHERE status = 1
ORDER BY create_time DESC, id DESC
LIMIT 20;
```

然后：

```
SELECT id, title, author, summary, content
FROM post
WHERE id IN (...20个id...);
```

第一步尽量走**覆盖索引 + LIMIT**，第二步再按主键取明细。
 这类优化本质是在减少排序阶段携带的数据量，避免把大字段带入排序或回表过早发生；官方 `filesort_summary` 里也能看到排序缓冲里到底是 `<sort_key, rowid>` 还是 `<sort_key, additional_fields>`。

------

### 5）少用 `ORDER BY` 表达式，必要时改成“可索引列”

例如不要：

```
ORDER BY DATE(create_time) DESC
```

而改成直接排序原列，或者增加一个生成列并索引它。
 因为表达式排序通常不能直接利用原索引。

------

### 6）复杂 join 排序时，先缩小结果集，再排

如果是：

```
SELECT ...
FROM a
JOIN b ...
JOIN c ...
WHERE ...
ORDER BY c.create_time DESC
LIMIT 20;
```

常见思路是先把过滤最强、排序字段所在的那部分结果集缩小，再 join 其余表。否则就很容易走 `Using temporary` / `Using filesort`。官方也明确提到，排序列来自 join 中不合适的位置时，会影响能否用索引排序。

------

### 7）让统计信息保持新鲜

大批量导入、冷热分布变化后，跑一下：

```
ANALYZE TABLE post;
```

因为优化器会基于索引基数和分布估计来选计划；统计不准时，它可能错误地觉得“扫描后排序”更便宜。

------

### 8）Hint 只在最后兜底

如果你确认优化器老是选错，可以用：

```
... FORCE INDEX FOR ORDER BY (idx_xxx)
```

或者 MySQL 8.x 的 `ORDER_INDEX` / `NO_ORDER_INDEX` hint。
 但这应该是最后手段，不是第一选择。

------

## 排序问题怎么排查

先看这三个东西就够用了：

```
EXPLAIN FORMAT=TREE ...
EXPLAIN ANALYZE ...
EXPLAIN FORMAT=JSON ...
```

`EXPLAIN` 可以看是否出现 `Using filesort`、`Using temporary`；`EXPLAIN ANALYZE` 会给真实执行时间；`FORMAT=JSON` / optimizer trace 还能看到 `filesort_summary`，包括临时文件数、峰值内存等。

再配合状态指标：

- `Sort_merge_passes`：看排序是否发生多轮 merge
- `Created_tmp_tables` / `Created_tmp_disk_tables`：看临时表是否频繁落盘

这些都是官方建议观察的点。

------

## 你可以直接记这个结论

对 MySQL 排序，最实用的判断顺序是：

**先争取“索引直接有序返回”**
 做不到，再争取**“小结果集 + LIMIT + 内存 filesort”**
 尽量避免**“大结果集 + 临时表 + 磁盘 filesort”**。

如果你愿意，我可以下一条直接拿一个你自己的 SQL，帮你判断它现在会走哪种排序方式，以及该建什么联合索引。

MySQL 是把返回数据排序完成后返回，还是只排序指定字段，再返回

我去核对一下 MySQL 官方文档里 `filesort` 的细节，因为这个问题的关键就在于：它并不总是“把整行都搬去排序”。

Thought for 28s

不是固定只会一种，**要分“排序阶段”和“返回阶段”看**。

**最终返回给客户端时，MySQL 会按 `ORDER BY` 要求的顺序返回整条结果行（也就是你 `SELECT` 出来的那些列）**；但在**内部排序阶段**，它**不一定把整行数据都拿去排序**。如果 `ORDER BY` 能直接用索引满足，就**没有额外排序阶段**，MySQL 直接按索引顺序读行并返回；如果不能，就会执行 `filesort`。`EXPLAIN` 里出现 `Using filesort` 就表示用了额外排序，没出现则通常是靠索引顺序完成的。

当发生 `filesort` 时，MySQL 排序缓冲里放的内容也**不总是整行**。官方给出的 `sort_mode` 有三种典型形式：`<sort_key, rowid>`、`<sort_key, additional_fields>`、`<sort_key, packed_additional_fields>`。这说明它可能只是**排序键 + 行指针**，也可能是**排序键 + 额外字段**，而不是无条件把整行完整搬进去排序。

你可以这样理解：

```
SELECT id, name, age
FROM user
ORDER BY create_time DESC;
```

有两种常见执行思路：

1. **走索引顺序**
    如果 `create_time` 上的索引正好能满足排序，MySQL 直接按索引顺序读，再把 `id, name, age` 返回，**不额外排序**。
2. **走 filesort**
    如果不能用索引，MySQL 会进入 `filesort`。这时它可能先排：
   - `create_time + rowid`，排完后再按排序结果去取 `id, name, age`
   - 或者 `create_time + additional_fields`，把部分需要返回的字段一起带着排，减少后续取行成本。

所以最准确的回答是：

**MySQL 最终是“按排好序的结果返回”；但内部并不一定“先把整条返回数据全部排序完再返回”，很多时候它只排“排序键 + 行定位信息”或“排序键 + 部分附加字段”。**

你做性能优化时，可以记这一条：

- 想让它最省事，就让 `ORDER BY` **尽量走索引**
- 一旦看到 `Using filesort`，就说明它进入了额外排序阶段
- `SELECT *` 往往更容易让排序后的取数成本变高，因为即使排序时没搬整行，后面也可能要回表拿更多列。官方文档也专门提醒，`SELECT *` 时优化器会权衡“按索引顺序读 + 取额外列”是否比“扫描后排序”更便宜。

把一句话压缩成面试版就是：

**返回时一定是有序结果；排序时不一定排整行，常见是排排序键，再配合 rowid 或 additional_fields。**

参考：

**https://zhuanlan.zhihu.com/p/618391862**
