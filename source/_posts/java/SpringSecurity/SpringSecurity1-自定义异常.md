---
title: SpringSecurity自定义异常
tags:
  - SpringSecurity
abbrlink: d6f2c8a8
date: 2021-01-28 15:00:00
---

**Spring Security** 中的异常主要分为两大类：一类是认证异常，另一类是授权相关的异常。

# AuthenticationException

`AuthenticationException` 是在用户认证的时候出现错误时抛出的异常。主要的子类如图：

![AuthenticationException.png](https://user-gold-cdn.xitu.io/2019/11/7/16e42fa6b14db7fe?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

根据该图的信息，系统用户不存在，被锁定，凭证失效，密码错误等认证过程中出现的异常都由 `AuthenticationException` 处理。

#  AccessDeniedException

`AccessDeniedException` 主要是在用户在访问受保护资源时被拒绝而抛出的异常。同 `AuthenticationException` 一样它也提供了一些具体的子类。如下图：

![AccessDeniedException.png](https://user-gold-cdn.xitu.io/2019/11/7/16e42fa6d6e875bf?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

`AccessDeniedException` 的子类比较少，主要是 `CSRF` 相关的异常和授权服务异常。

# Http 状态对认证授权的规定

**Http** 协议对认证授权的响应结果也有规定。

##  401 未授权状态

**HTTP 401 错误 - 未授权(Unauthorized)** 一般来说该错误消息表明您首先需要登录（输入有效的用户名和密码）。 如果你刚刚输入这些信息，立刻就看到一个 `401` 错误，就意味着，无论出于何种原因您的用户名和密码其中之一或两者都无效（输入有误，用户名暂时停用，账户被锁定，凭证失效等） 。总之就是认证失败了。其实正好对应我们上面的 `AuthenticationException` 。

## 403 被拒绝状态

**HTTP 403 错误 - 被禁止(Forbidden)**  出现该错误表明您在访问受限资源时没有得到许可。服务器理解了本次请求但是拒绝执行该任务，该请求不该重发给服务器。并且服务器想让客户端知道为什么没有权限访问特定的资源，服务器应该在返回的信息中描述拒绝的理由。一般实践中我们会比较模糊的表明原因。 该错误对应了我们上面的 `AccessDeniedException` 。

#  Spring Security 中的异常处理

我们在 **Spring Security** 实战干货系列文章中的 [自定义配置类入口 WebSecurityConfigurerAdapter](https://felord.cn/spring-security-httpsecurity.html) 一文中提到 `HttpSecurity` 提供的 `exceptionHandling()` 方法用来提供异常处理。该方法构造出 `ExceptionHandlingConfigurer` 异常处理配置类。该配置类提供了两个实用接口：



- **AuthenticationEntryPoint** 该类用来统一处理 `AuthenticationException` 异常
- **AccessDeniedHandler**  该类用来统一处理 `AccessDeniedException` 异常

我们只要实现并配置这两个异常处理类即可实现对 **Spring Security** 认证授权相关的异常进行统一的自定义处理。

### 4.1 实现 AuthenticationEntryPoint

以 `json` 信息响应。

```java
 import com.fasterxml.jackson.databind.ObjectMapper;
 import org.springframework.http.MediaType;
 import org.springframework.security.core.AuthenticationException;
 import org.springframework.security.web.AuthenticationEntryPoint;
 
 import javax.servlet.ServletException;
 import javax.servlet.http.HttpServletRequest;
 import javax.servlet.http.HttpServletResponse;
 import java.io.IOException;
 import java.io.PrintWriter;
 import java.util.HashMap;
 
 /**
  * @author dax
  * @since 2019/11/6 22:11
  */
 public class SimpleAuthenticationEntryPoint implements AuthenticationEntryPoint {
     @Override
     public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
 
         //todo your business
         HashMap<String, String> map = new HashMap<>(2);
         map.put("uri", request.getRequestURI());
         map.put("msg", "认证失败");
         response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
         response.setCharacterEncoding("utf-8");
         response.setContentType(MediaType.APPLICATION_JSON_VALUE);
         ObjectMapper objectMapper = new ObjectMapper();
         String resBody = objectMapper.writeValueAsString(map);
         PrintWriter printWriter = response.getWriter();
         printWriter.print(resBody);
         printWriter.flush();
         printWriter.close();
     }
 }
```



### 4.2 实现 AccessDeniedHandler

同样以 `json` 信息响应。

```java
 import com.fasterxml.jackson.databind.ObjectMapper;
 import org.springframework.http.MediaType;
 import org.springframework.security.access.AccessDeniedException;
 import org.springframework.security.web.access.AccessDeniedHandler;
 
 import javax.servlet.ServletException;
 import javax.servlet.http.HttpServletRequest;
 import javax.servlet.http.HttpServletResponse;
 import java.io.IOException;
 import java.io.PrintWriter;
 import java.util.HashMap;
 
 /**
  * @author dax
  * @since 2019/11/6 22:19
  */
 public class SimpleAccessDeniedHandler implements AccessDeniedHandler {
     @Override
     public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
         //todo your business
         HashMap<String, String> map = new HashMap<>(2);
         map.put("uri", request.getRequestURI());
         map.put("msg", "认证失败");
         response.setStatus(HttpServletResponse.SC_FORBIDDEN);
         response.setCharacterEncoding("utf-8");
         response.setContentType(MediaType.APPLICATION_JSON_VALUE);
         ObjectMapper objectMapper = new ObjectMapper();
         String resBody = objectMapper.writeValueAsString(map);
         PrintWriter printWriter = response.getWriter();
         printWriter.print(resBody);
         printWriter.flush();
         printWriter.close();
     }
 }
```



### 4.3 个人实践建议

其实我个人建议 **Http** 状态码 都返回 `200` 而将 401 状态在 元信息 `Map` 中返回。因为异常状态码在浏览器端会以 **error** 显示。我们只要能捕捉到 `401` 和 `403` 就能认定是认证问题还是授权问题。

### 4.4 配置

实现了上述两个接口后，我们只需要在 `WebSecurityConfigurerAdapter` 的 `configure(HttpSecurity http)` 方法中配置即可。相关的配置片段如下：

```java
 http.exceptionHandling().accessDeniedHandler(new SimpleAccessDeniedHandler()).authenticationEntryPoint(new SimpleAuthenticationEntryPoint())
```

## 总结

> https://juejin.cn/post/6844903988895154184
>
> https://ld246.com/article/1545318463746
>
> https://blog.csdn.net/qq_38225558/category_9395795.html
>
> 