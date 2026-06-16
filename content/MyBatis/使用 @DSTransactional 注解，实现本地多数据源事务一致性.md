+++
date = '2026-03-03T20:22:42+08:00'
draft = false
title = '使用 @DSTransactional 注解，实现本地多数据源事务一致性'
+++

之前在我的实习公司里，他们将一个金仓数据库的不同 schema 单独配置了数据库连接。我们通常使用 Spring `@Transactional` 只能管理一条数据库连接的配置。这就导致当跨 schema 调用时，使用 `@Transactional` 无法管理事务的一致性。

`@DSTransactional` 是 MP 提供的多数据源事务一致性方案。它能解决的核心是，**在同一个应用实例、同一个线程里，对多个数据源的操作，要么一起提交，要么一起回滚**。它本质是一个本地多数据源事务。

我们在 `application.yml` 配多数据源：

```yaml
spring:
  datasource:
    dynamic:
      primary: master
      datasource:
        master:
          url: jdbc:mysql://localhost:3306/db1?useSSL=false&serverTimezone=Asia/Shanghai
          username: root
          password: xxx
          driver-class-name: com.mysql.cj.jdbc.Driver
        slave:
          url: jdbc:mysql://localhost:3306/db2?useSSL=false&serverTimezone=Asia/Shanghai
          username: root
          password: xxx
          driver-class-name: com.mysql.cj.jdbc.Driver
```

可以使用 `@DS` 在 Mapper/Service 上切数据源。只需要在访问特定库的方法/类上标注：

```java
@DS("master")
public void writeMaster() { ... }

@DS("slave")
public void writeSlave() { ... }
```

也可以标在 mapper 接口或 service 类上。

当涉及到跨数据源事务的时候。使用 `@DSTransactional` 把跨两个数据源的业务逻辑包起来：

```java
import com.baomidou.dynamic.datasource.annotation.DS;
import com.baomidou.dynamic.datasource.annotation.DSTransactional;
import org.springframework.stereotype.Service;

@Service
public class OrderBizService {

    private final MasterMapper masterMapper;
    private final SlaveMapper slaveMapper;

    public OrderBizService(MasterMapper masterMapper, SlaveMapper slaveMapper) {
        this.masterMapper = masterMapper;
        this.slaveMapper = slaveMapper;
    }

    @DSTransactional
    public void createOrderAndLog(...) {
        // 1) 写主库
        masterMapper.insertOrder(...);

        // 2) 写另一个库
        slaveMapper.insertLog(...);

        // 3) 任意一步抛 RuntimeException，会触发两个库一起回滚
        // throw new RuntimeException("fail");
    }
}
```

这个 **跨库写必须发生在同一线程、同一调用链里**，否则这个本地事务上下文管不到。

它的核心是，**在同一线程里把多个数据源各自开启的本地事务捆绑成一个事务组**，最后统一 **commit** 或统一 **rollback**。

在进入 `@DSTransactional` 方法时，框架会在当前线程建立一个事务上下文/事务组。这个上下文会记录本次业务里用到的每个数据源对应的 **`TransactionStatus`** 或者连接/事务对象。当方法第一次切到某个数据源执行 SQL 时，它会：

- 为该数据源拿连接
- 开启该数据源自己的本地事务
- 把它加入事务组

如果正常结束。它会对事务组里的每个数据源事务执行 `commit`；

如果出现异常。它会对事务组里的每个数据源事务执行 `rollback`。
