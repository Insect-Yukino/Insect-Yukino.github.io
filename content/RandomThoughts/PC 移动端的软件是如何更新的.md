+++
date = '2026-01-03T19:20:00+08:00'
draft = false
title = 'PC 移动端的软件是如何更新的'
+++

> 比较好奇 PC、移动端上的软件或者系统是如何实时更新自身版本的。查阅了一下。

厂商是是通过**设备唯一标识 + 版本信息 + 服务器策略**在**设备主动连接服务器时**，动态判断该不该给你这个版本。

更新操作不是服务端的推送，而是设备主动询问

```http
POST /check-update
```

请求里可能包含

```json
{
  "deviceId": "a9f3c***",
  "model": "Pixel 7",
  "osVersion": "Android 14",
  "appVersion": "3.2.1",
  "region": "SG",
  "channel": "stable"
}
```

然后服务器会判断是否发放新版本

**决策逻辑**

```text
设备请求更新
  ↓
验证设备合法性
  ↓
读取当前版本
  ↓
匹配发布策略
  ↓
是否命中灰度
  ↓
返回结果
```

返回的内容可能如下

```json
{
  "hasUpdate": true,
  "version": "3.3.0",
  "force": false,
  "downloadUrl": "https://cdn.xxx.com/app-3.3.0.apk"
}
```
