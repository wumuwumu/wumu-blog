---
title: SpringSecurity3-Oauth2自定义返回
abbrlink: 2af70476
date: 2021-02-02 14:00:00
---

# token返回

## 扩展参数

```java

public class CusTokenConverter implements TokenEnhancer {


    @Override
    public OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
        Map<String, Object> additionalInformation = new LinkedHashMap<>();
        Map<String, Object> info = new LinkedHashMap<>();
        info.put("author", "wumu");
        info.put("user", SecurityContextHolder.getContext().getAuthentication().getPrincipal());
        additionalInformation.put("info", info);
        ((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(additionalInformation);
        return accessToken;
    }
}
```

## 重新格式化返回结果

```java

@RestController
@RequestMapping("/oauth")
public class OauthController {

    @Autowired
    private TokenEndpoint tokenEndpoint;

    @GetMapping("/token")
    public Map<String,Object> getAccessToken(Principal principal, @RequestParam Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
        return custom(tokenEndpoint.getAccessToken(principal, parameters).getBody());
    }

    @PostMapping("/token")
    public Map<String,Object> postAccessToken(Principal principal, @RequestParam Map<String, String> parameters) throws HttpRequestMethodNotSupportedException, HttpRequestMethodNotSupportedException {
        return custom(tokenEndpoint.postAccessToken(principal, parameters).getBody());
    }

    //自定义返回格式
    private Map<String,Object> custom(OAuth2AccessToken accessToken) {
        DefaultOAuth2AccessToken token = (DefaultOAuth2AccessToken) accessToken;
        Map<String, Object> data = new LinkedHashMap(token.getAdditionalInformation());
        data.put("accessToken", token.getValue());
        if (token.getRefreshToken() != null) {
            data.put("refreshToken", token.getRefreshToken().getValue());
        }
        return data;
    }

}
```

# 认证错误

```java

@ControllerAdvice
public class ExceptionConfig {


        @ResponseBody
        @ExceptionHandler(value = OAuth2Exception.class)
        public Map<String,Object> handleOauth2(OAuth2Exception e) {
            Map map = new HashMap<String,Object>();
            map.put("code",0);
            map.put("message",e.getMessage());
            return map;
        }


}
```



# 参考

> https://blog.csdn.net/u012702547/article/details/105804972
>
> https://juejin.cn/post/6857296054392471559
>
> https://blog.csdn.net/u013905744/article/details/100637224