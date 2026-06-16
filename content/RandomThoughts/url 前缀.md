+++
date = '2026-01-23T19:20:55+08:00'
draft = false
title = 'url 前缀'
+++

在实习过程中，第一次访问公司测试服务的接口。直接访问不到，看后端代码，路径没有问题。最后才发现微服务网关对请求进行了过滤，只有带有相应前缀的请求才能被访问。因此，下次如果遇到这种情况的话，往这方面想一下。一下是代码中的拦截器：

```java
@RequiredArgsConstructor
public abstract class ApiRequestFilter extends OncePerRequestFilter {

    protected final WebProperties webProperties;

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // 只过滤 API 请求的地址
        String apiUri = request.getRequestURI().substring(request.getContextPath().length());
        return !StrUtil.startWithAny(apiUri, webProperties.getAdminApi().getPrefix(), webProperties.getAppApi().getPrefix());
    }

}
```

在 spring mvc 的配置类中，我们也可以手动的给服务配置全局前缀，配置为：

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.addPathPrefix(
            "/api",
            clazz -> clazz.isAnnotationPresent(RestController.class)
        );
    }
}
```

也可以在 `yaml` 文件中配置

```yaml
server:
  servlet:
    context-path: /api
```
