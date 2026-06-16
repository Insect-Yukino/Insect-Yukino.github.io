+++
date = '2026-02-19T12:07:35+08:00'
draft = false
title = 'MP JOIN 时，分页 total 出错问题'
+++

### 1）原始查询 SQL

```sql
SELECT
  a.id,
  a.model_code,
  a.model_name,
  a.model_type,
  b.id   AS roamPathId,
  b.name AS roamPathName
FROM hbsw.iwater_v242.system_model_config a
LEFT JOIN hbsw.iwater_v242.system_model_camera_path b
  ON a.model_code = b.model_code
WHERE a.model_code IN (
  SELECT model_code
  FROM hbsw.iwater_v242.system_model_camera_path
  GROUP BY model_code
);
```

### 2）分页插件生成的统计总数 SQL（COUNT）

```sql
SELECT COUNT(*) AS total
FROM hbsw.iwater_v242.system_model_config a
WHERE a.model_code IN (
  SELECT model_code+
  FROM hbsw.iwater_v242.system_model_camera_path
  GROUP BY model_code
);
```

我的本意当然是查询返回所有 JOIN 的数据，但是 MyBatis-Plus 分页插件在优化 `count` SQL：它认为 `LEFT JOIN b` 对 `COUNT(*)` 没有影响，于是把 join 给去掉了，只对主表 `a` 做了 `count`。

所以 MP 生成的 `count` 就变成了：

```sql
SELECT COUNT(*) FROM a WHERE a.model_code IN (...)
```

而不是所期望的 **带 LEFT JOIN 的结构**。

##### 方案 1：关闭 `count` 优化

在发起分页前，关闭 MP 的 `count` SQL 优化：

```java
Page<ModelRespVO> page = new Page<>(current, size);
page.setOptimizeCountSql(false);   // 关键：不优化 count
page.setOptimizeJoinOfCountSql(false); // (有的版本叫这个)
```

不同 MyBatis-Plus 版本方法名略有差异：

- 常见是 `setOptimizeCountSql(false)`
- 有的版本是 `setOptimizeJoinOfCountSql(false)`

取决于项目的 MP 版本。

##### 方案 2：全局关闭 `join count` 优化

在 MyBatis-Plus 拦截器配置里关掉 `join` 优化：

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    PaginationInnerInterceptor pageInterceptor = new PaginationInnerInterceptor();
    pageInterceptor.setOptimizeJoin(false); // 关键：关闭 join 优化
    interceptor.addInnerInterceptor(pageInterceptor);
    return interceptor;
}
```

##### 方案 3：不让 MP 自动 count，你自己写 count（最可控）

如果希望 `count` 完全按自己写的 SQL 来：

- 关闭自动 `count`：`page.setSearchCount(false)`
- 自己写一个 `countXXX` mapper 方法执行带 join 的 count
- 手动 `page.setTotal(total)`

示例：

```java
page.setSearchCount(false);
List<ModelRespVO> records = mapper.selectXXX(page, req);
long total = mapper.countXXX(req);
page.setRecords(records);
page.setTotal(total);
```

XML 里 `countXXX` 就可以写成期望的 join 结构，例如：

```sql
SELECT COUNT(*)
FROM a
LEFT JOIN b ON ...
WHERE ...
```
