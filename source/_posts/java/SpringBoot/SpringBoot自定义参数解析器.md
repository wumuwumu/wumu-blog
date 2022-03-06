---
title: SpringBoot自定义参数解析器
tags:
  - SpringBoot
abbrlink: c3853fa8
date: 2021-04-26 13:00:00
---

对于如何自定义参数解析器，一个较推荐的方法是，先搞清楚springmvc接收到一个请求之后完整的处理链路，然后再来看在什么地方，什么时机，来插入自定义参数解析器，无论是从理解还是实现都会简单很多。遗憾的是，本篇主要目标放在的是使用角度，所以这里只会简单的提一下参数解析的链路，具体的深入留待后续的源码解析

# 参数解析链路

http请求流程图，来自 [SpringBoot是如何解析HTTP参数的](https://www.jianshu.com/p/bf3537334e76)

[![img](https://spring.hhui.top/spring-blog/imgs/190831/00.jpg)](https://spring.hhui.top/spring-blog/imgs/190831/00.jpg)

既然是参数解析，所以肯定是在方法调用之前就会被触发，在Spring中，负责将http参数与目标方法参数进行关联的，主要是借助`org.springframework.web.method.support.HandlerMethodArgumentResolver`类来实现

```java
/**
 * Iterate over registered {@link HandlerMethodArgumentResolver}s and invoke the one that supports it.
 * @throws IllegalStateException if no suitable {@link HandlerMethodArgumentResolver} is found.
 */
@Override
@Nullable
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

trueHandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
trueif (resolver == null) {
truetruethrow new IllegalArgumentException("Unknown parameter type [" + parameter.getParameterType().getName() + "]");
true}
truereturn resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```

上面这段核心代码来自`org.springframework.web.method.support.HandlerMethodArgumentResolverComposite#resolveArgument`，主要作用就是获取一个合适的`HandlerMethodArgumentResolver`，实现将http参数(`webRequest`)映射到目标方法的参数上(`parameter`)

所以说，实现自定义参数解析器的核心就是实现一个自己的`HandlerMethodArgumentResolver`

# HandlerMethodArgumentResolver

实现一个自定义的参数解析器，首先得有个目标，我们在get参数解析篇里面，当时遇到了一个问题，当传参为数组时，定义的方法参数需要为数组，而不能是List，否则无法正常解析；现在我们则希望能实现这样一个参数解析，以支持上面的场景

为了实现上面这个小目标，我们可以如下操作

## 自定义注解ListParam

定义这个注解，主要就是用于表明，带有这个注解的参数，希望可以使用我们自定义的参数解析器来解析；

```
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ListParam {
    /**
     * Alias for {@link #name}.
     */
    @AliasFor("name") String value() default "";

    /**
     * The name of the request parameter to bind to.
     *
     * @since 4.2
     */
    @AliasFor("value") String name() default "";
}
```

## 参数解析器ListHandlerMethodArgumentResolver

接下来就是自定义的参数解析器了，需要实现接口`HandlerMethodArgumentResolver`

```java
public class ListHandlerMethodArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(ListParam.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        ListParam param = parameter.getParameterAnnotation(ListParam.class);
        if (param == null) {
            throw new IllegalArgumentException(
                    "Unknown parameter type [" + parameter.getParameterType().getName() + "]");
        }

        String name = "".equalsIgnoreCase(param.name()) ? param.value() : param.name();
        if ("".equalsIgnoreCase(name)) {
            name = parameter.getParameter().getName();
        }
        String ans = webRequest.getParameter(name);
        if (ans == null) {
            return null;
        }

        String[] cells = StringUtils.split(ans, ",");
        return Arrays.asList(cells);
    }
}
```

# 注册

上面虽然实现了自定义的参数解析器，但是我们需要把它注册到`HandlerMethodArgumentResolver`才能生效，一个简单的方法如下

```java
@SpringBootApplication
public class Application extends WebMvcConfigurationSupport {

    @Override
    protected void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(new ListHandlerMethodArgumentResolver());
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class);
 
```