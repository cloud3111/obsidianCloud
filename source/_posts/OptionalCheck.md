---
title: 网络请求预检
tags:
  - Cors
categories: 编程
date: 2024-12-31 16:32:51
---

# 网络请求预检(OptionalCheck)

## 简介

- OPTIONS请求**即预检请求**，可用于检测服务器允许的http方法。当发起跨域请求时，由于安全原因，**触发一定条件时**浏览器会在正式请求之前**自动先发起OPTIONS请求**，即CORS预检请求，服务器若接受该跨域请求，返回一个通过响应, 浏览器才继续发起正式请求。
- 未配置允许`OPTIONS`请求，那么浏览器将收到一个**403 Forbidden**响应，表示服务器拒绝了该`OPTIONS`请求，`POST`请求的状态显示**CORS error**
- Access-Control-Max-Age: 跨域预检测的option请求有效期

## 分类

- 简单请求: 不需要预检验
  - 请求方法为: GET 和 POST和HEAD
  - 请求体类型 Content-Type 为: 除了application/json外所有
- 复杂请求: 通常需要发情预检验请求
  - 请求方式为: PUT 和 DELETE
  - 请求体类型 Content-Type 为: application/json
- 预检请求方式为特殊的: OPTION请求

## 参数

**简单请求**

- **Access-Control-Allow-Origin**
  - 必须, 表示源地址, 要么是一个`*`，表示接受任意域名的请求(如果 setAllowCredentials(true)，不能用 Access-Control-Allow-Origin: `*`，必须指定具体域名)
- **Access-Control-Allow-Credentials**
  - 非必须, 表示是否允许发送Cookie, Boolean类型
- **Access-Control-Expose-Headers**
  - 非必须, 请求头参数

**复杂请求**

- **Access-Control-Request-Method**
  - 预检请求必须: 询问请求方式是否允许
- **Access-Control-Request-Headers**
  - 预检请求必须, 询问请求头信息是否允许
- **Access-Control-Allow-Methods**
  - 预检响应必须: 得到允许的请求方式
- **Access-Control-Allow-Headers**
  - 预检响应必须: 得到允许的请求头信息
- **Access-Control-Allow-Credentials**
  - 预检响应非必须: 得到允许请求时携带Cookie
- **Access-Control-Max-Age**
  - 预检响应非必须: 单次预检生效时间, 超过需重新预检

## 跨域

- 什么是跨域: 通俗来说就是端对端访问了不同的 **地址 端口 路径** 的其中一种

- 解决方式: 
  - ​对于spring-Boot项目来说,只需要加一个注解 **@CrossOrigin**
  - 或者全局加一个过滤器
  - 如果 setAllowCredentials(true)，不能用 Access-Control-Allow-Origin: 通配符，必须指定具体域名
  - ​如果部署在Nginx上,需要在location块下加入如下参数
```nginx
  add_header Access-Control-Allow-Origin *;  # CORS 设置 跨域问题
  add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
  add_header Access-Control-Allow-Headers 'Origin, Content-Type, Accept';
```

## 例子

- 复杂请求的演示: OPTION预检请求格式   OPTION预检响应格式
```http
OPTIONS /cors HTTP/1.1 
Origin: http://ImOrigin.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header
Host: http://ImTarget.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0…
```

```http
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2.0.61 (Unix)   // 服务器信息
Access-Control-Allow-Origin: http://ImOrigin.com   // 也可以为*,允许全部 
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 0
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
```
- 后端例子: 要注意配置是否生效 , 网关就不能用普通的`implements GlobalFilter`过滤器, 要使用Spring WebFlux 内置的 CORS 过滤器, 执行顺序靠前

```java
/**  
 * spring层面的过滤器: addAllowedOriginPattern 框架特有的,不是请求头参数, 因为有了它才能使用setAllowCredentials  
 * 如果 setAllowCredentials(true)，不能用 Access-Control-Allow-Origin: *，必须指定具体域名 
 * 作为 Spring WebFlux 内置的 CORS 过滤器，CorsWebFilter 在请求进入 Gateway 之前 就已经处理了跨域请求（包括 OPTIONS 预检请求）。  
 * @author cloud_3111  
 * @since 2025-03-24  
**/
@Configuration  
@Slf4j  
public class GlobalCorsConfig {  
  
    private static final String ALL = "*";  
    private static final String MATCHING_PATH = "/**";  // 注意这里的斜杠  
    private static final Long MAX_AGE = 3600L;  
    public static final boolean TRUE = true;  
  
    @Bean  
    public CorsWebFilter corsWebFilter() {  
        log.info("*************跨域检测*************");  
        CorsConfiguration config = new CorsConfiguration();  
        // 这里仅为了说明问题，配置为放行所有域名，生产环境请对此进行修改:  
        config.addAllowedOriginPattern("*");  
        // 放行的请求头  
        config.addAllowedHeader(ALL);  
        // 放行的请求方式，主要有：GET, POST, PUT, DELETE, OPTIONS  
        config.addAllowedMethod(ALL);  
        // 暴露头部信息  
        config.addExposedHeader(ALL);  
        // 设置预检有效期: 3600秒(1小时)  
        config.setMaxAge(Duration.ofSeconds(MAX_AGE));  
        // 是否发送cookie  
        config.setAllowCredentials(TRUE);  
          
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();  
        source.registerCorsConfiguration(MATCHING_PATH, config);  
        return new CorsWebFilter(source);  
    }  
}
```
- 单体项目例子
```java
/**  
 * 跨域处理  
 * 已被废弃: Spring 提供的 CorsWebFilter（你的 GlobalCorsConfig）更早执行，在 HandlerMapping 级别拦截预检请求（OPTIONS）  
 * 网关就不能用普通的`implements GlobalFilter`过滤器, 要使用Spring WebFlux 内置的 CORS 过滤器, 执行顺序靠前  
 * @author cloud_3111  
 * @since 2025-03-19  
 *///@Component  
@Slf4j  
public class CorsFilter implements GlobalFilter, Ordered {  
  
    private static final String ALL = "*";  
    private static final String MAX_AGE = "3600L";  
    public static final String TRUE = "true";  
  
    @Override  
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {  
        log.info("跨域检测");  
  
        ServerHttpRequest request = exchange.getRequest();  
        ServerHttpResponse response = exchange.getResponse();  
  
        // 设置跨域响应头  
        HttpHeaders headers = response.getHeaders();  
        // 如果 setAllowCredentials(true)，不能用 Access-Control-Allow-Origin: *，必须指定具体域名  
        headers.add("Access-Control-Allow-Origin", "http://localhost:9000");  
        headers.add("Access-Control-Allow-Methods", ALL);  
        headers.add("Access-Control-Allow-Headers", ALL);  
        headers.add("Access-Control-Allow-Credentials", TRUE);  
        headers.add("Access-Control-Max-Age", MAX_AGE);  
  
        // 判断是否为预检验  
        if (request.getMethod() == HttpMethod.OPTIONS) {  
            // OPTIONS 预检请求 的目的是让浏览器询问服务器是否允许跨域,不需要手动加跨域头  
            log.info("预检请求");  
            response.setStatusCode(HttpStatus.OK);  
            return response.setComplete(); // 返回一个完整的 Mono<Void>        }  
        return chain.filter(exchange);  
    }  
  
    @Override  
    public int getOrder() {  
        // 最先被执行  
        return 0;  
    }  
}
```