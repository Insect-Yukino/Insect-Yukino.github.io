+++
date = '2025-08-04T21:09:13+08:00'
draft = false
title = 'SpringMvc自定义参数解析器'
+++

## HandlerMethodArgumentResolver — 方法参数解析器

```java
public interface HandlerMethodArgumentResolver {
	// 表示是否启用自定义参数解析器
	boolean supportsParameter(MethodParameter parameter);

	// 表示方法参数解析过程
	Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception;

}
```
