---
title: SpringCloud API聚合
tags:
  - SpringCloud
  - Swagger
abbrlink: 3cb9b8de
date: 2021-01-28 15:00:00
---

# 使用原本的Swagger

## 重写接口

```java

@RestController
public class SwaggerController {

    @Autowired
    private SwaggerService swaggerService;


    @ApiIgnore
    @RequestMapping(value = "/swagger-resources/configuration/security")
    ResponseEntity<SecurityConfiguration> securityConfiguration() {
        return new ResponseEntity<>(swaggerService.getSecurityConfiguration(), HttpStatus.OK);
    }

    @ApiIgnore
    @RequestMapping(value = "/swagger-resources/configuration/ui")
    ResponseEntity<UiConfiguration> uiConfiguration() {
        return new ResponseEntity<UiConfiguration>(swaggerService.getUiConfiguration(), HttpStatus.OK);
    }

    /**
     * 获取swagger服务列表，swagger页面自动请求
     *
     * @return list
     */
    @ApiIgnore
    @RequestMapping(value = "/swagger-resources")
    ResponseEntity<List<SwaggerResource>> swaggerResources() {
        return new ResponseEntity<>(swaggerService.getSwaggerResource(), HttpStatus.OK);
    }

    /**
     * 查询不包含跳过的服务的路由列表
     */
    @ApiIgnore
    @GetMapping("/v1/swaggers/resources")
    public ResponseEntity<List<SwaggerResource>> resources() {
        return new ResponseEntity<>(swaggerService.getSwaggerResource(), HttpStatus.OK);
    }


}

```

查询可以提供的接口服务，可以从注册中心中去查找，这里直接写固定的

```java

public interface SwaggerService {

    List<SwaggerResource> getSwaggerResource();

    UiConfiguration getUiConfiguration();

    SecurityConfiguration getSecurityConfiguration();


}
```

```java
@Component
public class SwaggerServiceImpl implements SwaggerService {

    @Autowired
    DiscoveryClient discoveryClient;

    @Override
    public List<SwaggerResource> getSwaggerResource() {
        List<SwaggerResource> resources = new LinkedList<>();
        SwaggerResource resource = new SwaggerResource();
        resource.setName("demo-user");
        resource.setSwaggerVersion("2.0");
        // 这里可以使用网关地址，获取自己手动请求，不然有跨域问题
        resource.setLocation("http://127.0.0.1:9000/user/v2/api-docs" );
        resources.add(resource);

        return resources;
    }

    @Override
    public UiConfiguration getUiConfiguration() {
        return new UiConfiguration(null);
    }

    @Override
    public SecurityConfiguration getSecurityConfiguration() {
        return new SecurityConfiguration(
                "", "unknown", "default",
                "default", "token",
                ApiKeyVehicle.HEADER, "token", ",");
    }
    

}
```

# knife4j

https://doc.xiaominfo.com/knife4j/resources/aggregation-introduction.html



# 参考

> https://juejin.cn/post/6854573219916201997
>
> 

