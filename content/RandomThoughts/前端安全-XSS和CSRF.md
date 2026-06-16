+++
date = '2025-08-07T10:28:27+08:00'
draft = false
title = '前端安全-XSS和CSRF'
+++

## XSS

### 概念

指跨站脚本攻击。攻击者通过向网站页面注入恶意脚本（一般是 JavaScript），通过恶意脚本对客户端网页进行篡改，达到窃取信息等目的，本质是数据被当作程序执行。

### HttpServletRequest

```java
public class XSSFilter implement Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws Exception {
        chain.doFilter(new XSSRequestWrapper((HttpServletRequest) request), response);
    }
} 

class XSSRequestWrapper extend HttpServletRequestWrapper {
    // HttpServletRequestWrapper 继承了 HttpServletRequest
    // 在 doFilter 方法中通过对传入的 request 进行包装，实现对原有 requset 操作方法的重写
    // 在重写逻辑中实现 XSS 逻辑的实现
}
```

## CSRF

### 概念

跨站请求伪造。是一种劫持受信任用户向服务器发送非预期请求的攻击方式。跨域指的是请求来源于其他网站，伪造指的是非用户自身的意愿。

### CSRF Token

实现`csr`f防御的一种方法
