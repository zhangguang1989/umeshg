---
layout:       post
title:        "spring-security和spring-cloud-gateway跨域问题"
author:       "Guang"
header-style: text
catalog:      true
tags:
    - Java
    - 跨域
---

spring-security和spring-cloud-gateway结合使用，可能出现跨域配置不生效的问题
# 解决办法
解决办法分两种

- 启用 Security 的跨域配置，手动指定 CorsConfigurationSource
- 配置一个 CorsConfigurationSource 的Bean，Security 自动完成注入
## 方法一
启用 Security 的跨域配置，手动指定 CorsConfigurationSource：
```java
@Bean
public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        // 异常处理配置
        http.
        // 其他配置
        ...
        // 重点在这儿！
        .and ()
        // 1. 开始跨域配置
        .cors ()
        // 2. 指定一个 CorsConfigurationSource`
        .configurationSource (corsConfigurationSource())
        // 其他配置
        .and ().csrf ().disable ();
        return http.build ();
}
// 跨域配置
public CorsConfigurationSource corsConfigurationSource() {
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource (new PathPatternParser ());
    CorsConfiguration corsConfig = new CorsConfiguration ();

    // 允许所有请求方法
    corsConfig.addAllowedMethod ("*");
    // 允许所有域，当请求头
    corsConfig.addAllowedOrigin ("*");
    // 允许全部请求头
    corsConfig.addAllowedHeader ("*");
    // 允许携带 Authorization 头
    corsConfig.setAllowCredentials (true);
    // 允许全部请求路径
    source.registerCorsConfiguration ("/**", corsConfig);
    return source;
}
```
## 方法二
上面是手动指定 CorsConfigurationSource，其实只要暴露一个 CorsConfigurationSource 的Bean，Spring Security 会自动完成跨域配置：
```java
@Bean
    public CorsConfigurationSource corsConfigurationSource() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource (new PathPatternParser ());
        CorsConfiguration corsConfig = new CorsConfiguration ();

        // 允许所有请求方法
        corsConfig.addAllowedMethod ("*");
        // 允许所有域，当请求头
        corsConfig.addAllowedOrigin ("*");
        // 允许全部请求头
        corsConfig.addAllowedHeader ("*");
        // 允许携带 Authorization 头
        corsConfig.setAllowCredentials (true);
        // 允许全部请求路径
        source.registerCorsConfiguration ("/**", corsConfig);
        return source;
    }
```
# 探究原因
## 为什么需要对Sping security进行单独的跨域配置？
首先必须明确几点：
- Spring cloud gateway的跨域都是通过CorsWebFilter这个过滤器来完成的。上面配置CorsConfigurationSource，也无非是让Spring Security利用CorsConfigurationSource自动构建出一个CorsWebFilter添加到它的过滤链中。
- 对于跨域的请求，浏览器会先发送一个options方法的跨域预检请求，询问服务器是否允许浏览器的跨域请求，如果浏览器响应了正确的响应头，浏览器才能继续发送跨域请求，否则就会报跨域错误。  

我们知道Spring Security也是基于过滤器的，既然是过滤器那就会涉及到一个先后的问题，事实证明 gateway 的CorsWebFilter是在Spring Security的过滤链之后的。
一个options方法的跨域预检请求来到Spring Security的过滤链，Spring Security一看这个options请求没有携带用户凭证的Authorization头、Cookie或者token参数等信息，直接把这个请求给拦了下来，浏览器收不到正确的响应就会报跨域错误。所以如果使用了Sping security，就需要为Sping security配置跨域，让Sping security添加CorsWebFilter到自己的过滤链中，防止跨域预检请求被直接拒了。

