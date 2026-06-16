+++
date = '2025-12-07T19:42:09+08:00'
draft = false
title = 'Nacos 配置'
+++

nacos首先分为客户端和服务端。启动nacos后只需要在客户端配置nacos的配置信息，让客户端能找到服务端就可以了

```yaml
spring:
  cloud:
    nacos:
      server-addr: localhost:8848 # nacos注册中心
      username: nacos
      password: nacos
      discovery:
        server-addr: localhost:8848
        namespace: ed3bce02-9990-48fc-8c80-475551958fbf
        group: DEFAULT_GROUP
      config:
        server-addr: localhost:8848
        namespace: ed3bce02-9990-48fc-8c80-475551958fbf
        file-extension: yaml
  config:
    import:
      - nacos:${spring.application.name}.yaml # 导入本服务的配置
      - nacos:shared-spring.yaml?group=SHARED_GROUP # 共享spring配置
      - nacos:shared-redis.yaml?group=SHARED_GROUP # 共享redis配置
      - nacos:shared-mybatis.yaml?group=SHARED_GROUP # 共享mybatis配置
      - nacos:shared-logs.yaml?group=SHARED_GROUP # 共享日志配置
      - nacos:shared-feign.yaml?group=SHARED_GROUP # 共享feign配置
      - nacos:shared-mq.yaml?group=SHARED_GROUP # 共享mq配置
      - nacos:shared-xxljob.yaml?group=SHARED_GROUP # 共享xxljob配置
```

# SkyWalking 配置

skywalking 大致分为服务端和客户端，客户端需要在 JVM 上挂载 Java Agent

```bash
-javaagent:D:\JavaLearning\component\skywalking-agent\skywalking-agent.jar 
-Dskywalking.agent.service_name=user-service 
-Dskywalking.agent.collector_backend_services=localhost:11800
```

第一个参数是它的agent jar包

第二个参数是web端界面显示的服务名称

第三个参数是客户端于服务端的通信地址

# Seata 配置

seata 也是大致分为服务端和客户端。它首先需要你在服务端本地文件中配置好注册中心，配置中心，指定从配置中心加载的配置文件名称等主要配置。

```yaml
seata:
  config:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: ed3bce02-9990-48fc-8c80-475551958fbf
      group: SEATA_GROUP
      username: nacos
      password: nacos
      context-path:
      data-id: seataServer.yaml
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      namespace: ed3bce02-9990-48fc-8c80-475551958fbf
      cluster: default
      username: nacos
      password: nacos
      context-path:
```

同时，需要在服务端加载的配置文件中配置客户端的事务分组名

```yaml
service:
  vgroup-mapping:
    tj_tx_group: default
```

即使客户端所在服务配置了nacos访问授权信息，仍需在其seata的配置中重申，并且指定其事务分组信息

```yaml
service:
  vgroup-mapping:
    tj_tx_group: default
registry:
  type: nacos
  nacos:
    application: seata-server
    server-addr: 127.0.0.1:8848
    group: SEATA_GROUP
    namespace: ed3bce02-9990-48fc-8c80-475551958fbf
    cluster: default
    username: nacos
    password: nacos
```

大致是这样的配置。seata的事务分组的名称需要按照事务界限分配
