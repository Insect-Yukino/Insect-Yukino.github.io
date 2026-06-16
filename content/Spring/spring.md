+++
date = '2025-12-22T22:02:38+08:00'
draft = false
title = 'Spring'
+++

**Spring 的核心思想是 IoC 和 AOP**：

- **IoC**（控制反转）：它把对象的创建和依赖关系交给 Spring 容器管理，不需要开发人员手动管理对象。
- **AOP**（面向切面编程）：将与业务无关的功能，例如日志、事务、安全等从业务中剥离出来，降低耦合。

## IoC 的实现方式

**Spring IoC 主要是通过反射、依赖注入、依赖查找等机制实现。**

- 反射：它为 Spring 提供了动态加载类、创建对象实例以及调用对象方法等机制。Spring 以此实现了灵活的对象实例化和管理。
- 依赖注入 & 依赖查找：这是控制反转思想的两种实现。**依赖注入**主要是依靠 `@Autowired`、`@Component` 等注解，自动实现对象的创建和对象之间依赖的管理；依赖查找主要是依靠 `ApplicationContext` 容器，手动查找此时需要的对象实例。

> `ApplicationContext` 是 IoC 容器的基本形式。是 Spring 中最重要的组件之一。

### 依赖注入的几种方式

- **构造器注入**

  ```java
  @Service
  public class OrderService {
  
      private final UserService userService;
  	
      public OrderService(UserService userService) {
          this.userService = userService;
      }
  }
  ```

  如果只有一个构造器，`@Autowired` 可省略。并且，构造器注入可以配合 Lombok 提供的 `@RequiredArgsConstructor` 注解使用

  ```java
  @RequiredArgsConstructor
  @Service
  public class OrderService {
      private final UserService userService;
  }
  ```

  这里 `@RequiredArgsConstructor` 注解会为所有为初始化的 `final` 字段生成一个构造器，然后 Spring 发现只有一个构造器，就会为其进行构造注入。

  > - `@RequiredArgsConstructor` 会为所有 `final` 字段和未初始化的 `@NonNull` 字段生成构造器参数，而不是仅仅针对 `final` 字段。它的设计目的是保证对象在构造阶段完成必要依赖的注入。
  >
  > - 需要注入的 Spring Bean 用 `final` 修饰，也有其语义，这是为了保证 **依赖不可变**。

- **Setter 注入**

  ```java
  @Service
  public class OrderService {
  
      private UserService userService;
  
      @Autowired
      public void setUserService(UserService userService) {
          this.userService = userService;
      }
  }
  ```

- **字段注入**

  ```java
  @Service
  public class OrderService {
  
      @Autowired
      private UserService userService;
  }
  ```

- **方法参数注入**

  ```java
  @Configuration
  public class AppConfig {
  
      @Bean
      public OrderService orderService(UserService userService) {
          return new OrderService(userService);
      }
  }
  ```

**Spring 官方大力推荐使用构造器方式注入 Bean，是为了保证 Bean 在创建完成后依赖完整、状态不可变、线程安全，并且能够快速失败、提升可测试性和代码可维护性。**

### Bean 的生命周期

我们在享受着控制反转的便利时，是否考虑过 Bean 的生命周期是什么样的。

Bean 的生命周期大致可以分为**实例化**、**属性赋值**、**初始化**、**使用**和**销毁**。

- 实例化是执行 Bean 的构造器
- 属性赋值主要是执行依赖注入
- 初始化主要是执行业务层的逻辑

Spring 在 Bean 的整个生命周期中提供了大量的扩展点，如下：

1. 实例化 Bean（构造器）
2. 属性填充（依赖注入）
3. **Aware 接口回调**
4. **BeanPostProcessor 前置处理**
5. 初始化（@PostConstruct / InitializingBean）
6. **BeanPostProcessor 后置处理**
7. Bean 就绪
8. 容器关闭
9. 销毁（@PreDestroy / DisposableBean）

#### Aware 接口

**Aware 接口用于让 Bean 感知 Spring 容器及其运行环境，通过生命周期回调的方式注入容器相关对象。**例如：

- `BeanNameAware`，让 Bean 可以拿到自身在容器中注册的名称

  ```java
  public class MyBean implements BeanNameAware {
      @Override
      public void setBeanName(String name) {
          this.beanName = name;
      }
  }
  ```

- `BeanFactoryAware` 可以让 Bean 获取 `BeanFactory` 从而动态获取其他 Bean

  ```java
  public class MyBean implements BeanFactoryAware {
      @Override
      public void setBeanFactory(BeanFactory beanFactory) {
          this.beanFactory = beanFactory;
      }
  }
  ```

- `ApplicationContextAware` 最常用，可以获取整个 Spring 容器上下文

  ```java
  public class MyBean implements ApplicationContextAware {
      @Override
      public void setApplicationContext(ApplicationContext ctx) {
          this.context = ctx;
      }
  }
  ```

#### BeanPostProcessor 

是 Spring 提供的最重要的扩展点之一。**它会在 Bean 初始化前后，对 Bean 进行统一的加工、增强或替换，是 Spring AOP、注解解析、自动注入等机制的基础。**

接口如下：

```java
public interface BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

上述两个方法的参数、返回值含义相同。`bean` 就是正在初始化的 Bean 实例，`beanName` 表示 Bean 在容器中的名称。返回值是返回的 Bean 实例。

**因此 `BeanPostProcessor` 可以在初始化前后对该 Bean 进行加工。最终的返回值决定了容器最终使用的 Bean 对象。AOP 代理通常在 `postProcessAfterInitialization` 中创建并返回。**

### 循环依赖问题

介绍完 Bean 的注入方法和生命周期，现在我们来看一下 Bean 在依赖注入的时候可能面临的问题——**循环依赖**。

以下情况下就会发生循环依赖

```java
@Service
public class ServiceA {
    @Autowired
    private ServiceB b; // 访问修饰符问题
}
---------------------------------------
@Service 
public class ServiceB {
    @Autowired
    private ServiceA a;
}
```

**上面介绍的所有依赖注入方式都可能会发生循环依赖问题，但是 Spring 只解决了单例模式下 Setter 注入和字段注入产生的循环依赖问题。构造器循环依赖通常会直接创建失败。**

> 这里补充一下 Spring Bean 的作用域
>
> Spring 内置常用作用域如下：
>
> | 作用域    | 关键字        | 说明                        |
> | --------- | ------------- | --------------------------- |
> | **单例**  | `singleton`   | 默认，一个容器一个实例      |
> | **原型**  | `prototype`   | 每次获取一个新实例          |
> | 请求      | `request`     | 每个 HTTP 请求一个实例      |
> | 会话      | `session`     | 每个 HTTP Session 一个实例  |
> | 应用      | `application` | Web 应用级别单例            |
> | WebSocket | `websocket`   | 每个 WebSocket 会话一个实例 |
>
> - 我们在创建 Bean 时，因为 `singleton` 是默认的，所以 `@Scope("singleton")` 可以省略
>
>   ```java
>   @Component
>   @Scope("singleton") // 可省略
>   public class UserService {}
>   ```
>
>   **全局只有一个 Bean 时，该 Bean 的生命周期就会由 Spring 容器管理。**
>
>   **由于全局共享同一个 Bean，因此我们要保证该 Bean 是无状态，以此来保证线程安全问题**
>
> - **如果是 `prototype` Bean，程序每次 `getBean()` 时都会创建一个新实例，Spring 只负责创建，不负责销毁**
>
>   ```java
>   @Component
>   @Scope("prototype") // 此处不能省略
>   public class Task {}
>   ```
>
> **`@Scope` 注解就是用来指定 Bean 作用域的**

#### 三级缓存

Spring 是如何解决循环依赖问题的？标题已经暴露了答案

Spring 维护了三个重要缓存（Map）：

- **一级缓存**：用来存放完全初始化好的、可用的 Bean 实例
- **二级缓存**：存放着提前暴露的 Bean 的原始对象引用或早期代理对象引用。专门用来处理循环依赖。
- **三级缓存**：存放的是 Bean 的 `ObjectFactory` 工厂对象。这是解决循环依赖和 AOP 代理协同的关键。

#### Spring 为什么要用 3 级缓存解决依赖问题

Spring 主要是依靠 `ObjectFactory` 生产出符合要求的 Bean。

## AOP 的实现方式

Spring AOP 的实现主要依赖于**动态代理技术**。

动态代理是在运行时动态生成代理对象，Spring AOP 支持两种动态代理：

- **基于 JDK 的动态代理**：JDK 动态代理是 Java 原生提供的动态代理，主要使用 `java.lang.reflect.Proxy` 和 `java.lang.reflect.InvocationHandler` 实现。这种方式需要代理的类实现一个或多个接口。
- **基于 CGLIB 的动态代理**：是 CGLIB 第三方代码生成库，通过继承的方式实现代理。

以上两种方式的缺陷

### Spring AOP 的应用

#### 事务管理

#### 日志记录

#### 权限校验
