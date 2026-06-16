+++
date = '2025-10-11T16:50:27+08:00'
draft = false
title = 'URL 设计'
+++

刚学的时候 Java 后端的 url 地址一般都是随便写的，像 `/getUser`、`/saveUser` 等。但是这些都是不标准的写法。

URL 是有设计规范的。

> **URL** 表达的是 **资源（Resource）**而不是 **行为（Action）**。用 **HTTP 动词** 表达操作类型，用 **路径层级** 表达资源关系

### URL 命名规范

- 字母全部小写，不能使用大写字母
- 如果是多个单词组合，中间要使用 `-` 分隔单词
- URL 表示资源集合，所以要使用复数名词
- 不要再路径中使用动词，动词一般是用 HTTP 方法表达
- URL 路径要体现资源层级关系，子资源从属父资源
- 路径中可以体现版本控制

```markdown
[版本] / [业务领域] / [资源名] / [资源ID] / [子资源] / [动作]
```

### HTTP 方法与语义对应表

| 方法     | 动作                 | 示例                                                  |
| -------- | -------------------- | ----------------------------------------------------- |
| `GET`    | 获取资源             | `GET /api/v1/users`                                   |
| `POST`   | 创建资源 / 复杂查询  | `POST /api/v1/devices` / `POST /api/v1/devices/query` |
| `PUT`    | 更新资源（整体更新） | `PUT /api/v1/devices/{id}`                            |
| `PATCH`  | 部分更新             | `PATCH /api/v1/devices/{id}`                          |
| `DELETE` | 删除资源             | `DELETE /api/v1/devices/{id}`                         |

### 层级结构设计原则

#### 单资源

| 类型     | URL 示例                 | 操作示例                                  |
| -------- | ------------------------ | ----------------------------------------- |
| 集合资源 | `/api/v1/users`          | `GET` 获取列表，`POST` 新增               |
| 单个资源 | `/api/v1/users/{userId}` | `GET` 获取详情，`PUT` 更新，`DELETE` 删除 |

#### 子资源（属于某个父资源）

> 表达从属关系，例如设备的报警记录、参数等。

| 场景           | URL 示例                                     |
| -------------- | -------------------------------------------- |
| 设备报警记录   | `/api/v1/devices/{deviceId}/alarm-records`   |
| 设备温湿度参数 | `/api/v1/devices/{deviceId}/params/temp-hum` |
| 用户日志       | `/api/v1/users/{userId}/logs`                |

#### 动作类接口（非纯 CRUD）

> 如果操作不是简单的 CRUD，而是特殊动作（如激活、导出、测试连接），
>  用动词放在子路径上表达动作。

| 动作       | URL 示例                              | 方法   |
| ---------- | ------------------------------------- | ------ |
| 激活设备   | `/api/v1/devices/{deviceId}/activate` | `POST` |
| 导出日志   | `/api/v1/user/logs/export`            | `GET`  |
| 测试MQ连接 | `/api/v1/system/mqtt/test-connection` | `POST` |

### 查询参数设计

一般来说查询操作是使用 GET 操作，这时它的查询参数一般会使用 `?param1=value1&param2=value2` 缀在路径后面。但是，当有大量查询参数时，这样的形式就不适合了，这时就可以使用 POST 查询，查询参数由 `Json` 请求体携带

#### 简单查询

```bash
GET /api/v1/alarm-records?startTime=2025-10-01T00:00:00&endTime=2025-10-11T23:59:59&page=1&pageSize=20
```

#### 复杂查询

```bash
POST /api/v1/alarm-records/query
Content-Type: application/json

{
  "deviceIds": [1, 2, 3],
  "timeRange": { "start": "2025-10-01T00:00:00", "end": "2025-10-11T23:59:59" },
  "levels": ["high", "medium"],
  "sort": { "field": "timestamp", "order": "desc" },
  "page": 1,
  "pageSize": 20
}
```
