---
title: SpringCloudFeign学习2-自定义负载均衡规则
tags:
  - SpringCloud
abbrlink: 287249d4
date: 2021-02-19 11:00:00
---

# 实战

```java
public class GrayMetadataRule extends AbstractLoadBalancerRule {
    Logger logger = LoggerFactory.getLogger(getClass());
    @Autowired
    private NacosDiscoveryProperties nacosDiscoveryProperties;
    @Autowired
    private NacosServiceManager nacosServiceManager;


    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {

    }

    @Override
    public Server choose(Object key) {
        String clusterName = this.nacosDiscoveryProperties.getClusterName();
        String group = this.nacosDiscoveryProperties.getGroup();
        DynamicServerListLoadBalancer loadBalancer = (DynamicServerListLoadBalancer)this.getLoadBalancer();
        String name = loadBalancer.getName();
        NamingService namingService = this.nacosServiceManager.getNamingService(this.nacosDiscoveryProperties.getNacosProperties());
        List<Instance> instances = null;
        try {
            instances = namingService.selectInstances(name, group, true);
        } catch (NacosException e) {
            e.printStackTrace();
        }
        if(instances == null || instances.size() == 0){
            logger.warn("没有相关服务 {}",name);
            return null;
        }
        System.out.println("找到相关服务");
        return new NacosServer(instances.get(0));
    }
}
```

# 参考

> https://www.cnblogs.com/ITPower/p/13353248.html
>
> https://blog.csdn.net/forezp/article/details/74820899
>
> https://blog.didispace.com/springcloud-sourcecode-ribbon/
>
> https://www.cnblogs.com/rickiyang/p/11802465.html

