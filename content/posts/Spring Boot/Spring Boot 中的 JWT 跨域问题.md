---
title: Spring Boot 中的 JWT 跨域问题
date: 2020-06-05 21:02:00
tags: 
- Security
- Auth
- Spring Boot
categories: 
- Spring Boot
---

# 主要问题

前后端分离后，前端使用 Ajax 进行请求，存在一些跨域的问题。

基于 Spring Boot 的后端通过给 Controlle类或其中的方法添加 `@CrossOrigin` 注解来解决跨域问题：

```java
@CrossOrigin
@RequestMapping("/user/")
public class UserController {
	...
}
```

添加该注解之后，可以通过匿名访问的接口都没有跨域问题了，而需要通过 JWT 验证的接口仍然存在跨域问题。

其中，**解决问题的关键**在于，浏览器会在发送 Ajax 请求之前发送一个预请求，确认当前的接口是不是有效的接口，此时的请求方式是 OPTIONS 的请求方式。

因此，JWT 的过滤器需要先判断该请求是否为预请求，如果是则需要给返回的响应头中添加跨域相关的信息；如果不是，则按照一般接口进行 JWT 验证。

```java
public class AuthFilter extends OncePerRequestFilter {

  	......

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
    
        // 处理浏览器的预请求
        if (request.getMethod().equals("OPTIONS")) {
            response.setHeader("Access-Control-Allow-Origin", "*");
            response.setHeader("Access-Control-Allow-Methods", "POST,GET,PUT,OPTIONS,DELETE");
            response.setHeader("Access-Control-Max-Age", "3600");
            response.setHeader("Access-Control-Allow-Headers", "Origin,X-Requested-With,Content-Type,Accept,Authorization,token");
            return;
        }
        
        // 验证 JWT
        ......
    }
}
```

