+++
date = '2025-11-16T21:14:40+08:00'
draft = false
title = 'mvc在异步时的请求行为和拦截器的行为'
+++

Spring MVC 支持异步请求，在处理比较耗时的 IO 操作时，可以选择使用异步操作。使用的时候只使用 `Callable` 或 `DeferredResult` 作为控制器中方法的返回值。

当 `controller` 返回的是 `Callable` 的时候，`springmvc` 就会启动一个线程将 `Callable` 交给 `TaskExecutor` 去处理，然后 `DispatcherServlet` ，还有所有的 `spring` 拦截器都退出主线程，然后把 `response` 保持打开的状态，当 `Callable` 执行结束之后，`springmvc `就会重新启动分配一个 `request `请求，然后 `DispatcherServlet` 就重新调用和处理 `Callable` 异步执行的返回结果， 然后返回视图

这时注意，`spring` 的拦截器就会有重复执行的操作

`WebAsyncTask `

DeferredResult 是事件驱动异步（长时间等待外部异步结果）。
 WebAsyncTask 是后台线程池异步（本地执行耗时任务）。
