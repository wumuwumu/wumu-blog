---
title: rocketmq源码运行
date: 2022-2-13 16:25:00
tags:
- rocketmq
- java
- mq
---

### 1. 基本架构

`RocketMQ`架构上主要分为四部分，如下图所示:

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d7893db87d2415297dc76cf07a9da25~tplv-k3u1fbpfcp-watermark.awebp)

- `Producer`：消息发布的角色，支持分布式集群方式部署。`Producer`通过`MQ`的负载均衡模块选择相应的`Broker`集群队列进行消息投递，投递的过程支持快速失败并且低延迟。

- `Consumer`：消息消费的角色，支持分布式集群方式部署。支持以`push`推，`pull`拉两种模式对消息进行消费。同时也支持集群方式和广播方式的消费，它提供实时消息订阅机制，可以满足大多数用户的需求。

- `NameServer`：`NameServer`是一个非常简单的`Topic`路由注册中心，其角色类似`Dubbo`中的`zookeeper`，支持`Broker`的动态注册与发现。主要包括两个功能：
  
  - `Broker`管理，`NameServer`接受`Broker`集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查`Broker`是否还存活；
  - 路由信息管理，每个`NameServer`将保存关于`Broker`集群的整个路由信息和用于客户端查询的队列信息。然后`Producer`和`Conumser`通过`NameServer`就可以知道整个`Broker`集群的路由信息，从而进行消息的投递和消费。
  
  `NameServer`通常也是集群的方式部署，各实例间相互不进行信息通讯。`Broker`是向每一台`NameServer`注册自己的路由信息，所以每一个`NameServer`实例上面都保存一份完整的路由信息。当某个`NameServer`因某种原因下线了，`Broker`仍然可以向其它`NameServer`同步其路由信息，`Producer`,`Consumer`仍然可以动态感知`Broker`的路由的信息。

- `BrokerServer`：`Broker`主要负责消息的存储、投递和查询以及服务高可用保证，为了实现这些功能，`Broker`包含了以下几个重要子模块：
  
  - `Remoting Module`：整个`Broker`的实体，负责处理来自`clients`端的请求。
    - `Client Manager`：负责管理客户端(`Producer`/`Consumer`)和维护`Consumer`的`Topic`订阅信息
    - `Store Service`：提供方便简单的API接口处理消息存储到物理硬盘和查询功能。
    - `HA Service`：高可用服务，提供`Master Broker` 和 `Slave Broker`之间的数据同步功能。
    - `Index Service`：根据特定的`Message key`对投递到`Broker`的消息进行索引服务，以提供消息的快速查询。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a1d8173de75499f9b46a7aa7091da57~tplv-k3u1fbpfcp-watermark.awebp)

## 2. 获取源码

rocketMq项目的`github`仓库为[github.com/apache/rock…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fapache%2Frocketmq.git "https://github.com/apache/rocketmq.git")，由于网络原因，我们并不会直接使用`github`仓库，而是将其导入到`gitee`上，只需在`gitee`创建新仓库时，选择导入已有仓库即可：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/590692126e494bb3bf6afe811d6c4770~tplv-k3u1fbpfcp-watermark.awebp)

导入到`gitee`后，就可以进行`checkout`了，本文对应的gitee仓库为[gitee.com/funcy/rocke…](https://link.juejin.cn?target=https%3A%2F%2Fgitee.com%2Ffuncy%2Frocketmq.git "https://gitee.com/funcy/rocketmq.git")。

`checkout`源码到本地后，默认是`master`分支，本人习惯基于`tag`创建自己的分支，然后在自己的分支上进行分析，`rocketMq`的`tag`如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1785db1bb7d040a195a449d3f5a79389~tplv-k3u1fbpfcp-watermark.awebp)

最新版本是`4.8.0`，我们将基于此tag创建新分支，使用的命令如下：

```sh
# 切换到 rocketmq-all-4.8.0
git checkout rocketmq-all-4.8.0
# 基于 rocketmq-all-4.8.0 创建自己的分析，名称为 rocketmq-all-4.8.0-LEARN
git checkout -b rocketmq-all-4.8.0-LEARN
# 将 rocketmq-all-4.8.0-LEARN 分支推送到远程仓库
git push -u origin rocketmq-all-4.8.0-LEARN
复制代码
```

接下来，我们所有的操作都是在`rocketmq-all-4.8.0-LEARN`分支上进行了。

### 3. 本地启动

拿到代码后，我们就开始进行本地启动了，没错，就是在idea中进行启动。

#### 3.1 复制`conf`目录

在启动项目前，我们需要进行一些配置，`rocketMq`项目的配置文件位于`rocketmq/distribution`模块下的`conf`目录中，直接整个复制到`rocketmq`目录下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e5b2dac07b043769e47ceec7506ba82~tplv-k3u1fbpfcp-watermark.awebp)

也不需要改动，复制出来就行了，这些配置的内容后面分析源码时再讲解吧。

#### 3.2 启动`nameServer`

`nameServer`的主类为`org.apache.rocketmq.namesrv.NamesrvStartup`：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/840bae7379c54826ae0acff203bdf51d~tplv-k3u1fbpfcp-watermark.awebp)

如果我们直接运行`main()`方法，会报错：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1dd9d3189a3c487a8a7769d34fc73e8b~tplv-k3u1fbpfcp-watermark.awebp)

报错信息已经很明确了，需要我们配置`ROCKETMQ_HOME`目录，我们在`idea`中进行配置即可：

打开配置界面：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/724bce8485e6452db2e7d5577c9e98fc~tplv-k3u1fbpfcp-watermark.awebp)

填写`ROCKETMQ_HOME`配置：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f6b159a7122a422eade6ee30a7a8cfeb~tplv-k3u1fbpfcp-watermark.awebp)

这里我填写的是`ROCKETMQ_HOME=/Users/chengyan/IdeaProjects/myproject/rocketmq`，这个`ROCKETMQ_HOME`路径就是`conf`文件夹所在的目录。

填写好后，就可以启动了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50dc93afe5c042c39d437315668648ef~tplv-k3u1fbpfcp-watermark.awebp)

#### 3.3 启动`broker`

`broker`的主类为`org.apache.rocketmq.broker.BrokerStartup`，启动方式与`nameServer`很相似，启动前也要配置`ROCKETMQ_HOME`路径：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ddcfda6e8349489fac00db4df6e2ec77~tplv-k3u1fbpfcp-watermark.awebp)

相比于`nameServer`，这里多配置了启动参数：

```
-n localhost:9876 autoCreateTopicEnable=true
复制代码
```

这个启动参数是指定`nameServer`的地址，以及开启自动创建`topic`的功能。

配置完成之后就可以启动了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b68e17980f264a4d9994000a92bb3dd9~tplv-k3u1fbpfcp-watermark.awebp)

#### 3.4 启动管理后台

`rocketMq`的管理后台在另一个仓库[github.com/apache/rock…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fapache%2Frocketmq-externals "https://github.com/apache/rocketmq-externals")，除了后台，这个仓库还包含了许多的其他模块：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0da6a8741ca4e789c3338d23528249e~tplv-k3u1fbpfcp-watermark.awebp)

我们并不需要分析这个项目，源码本可以不必下载，但我在找这个项目的`release`版本时，发现并没有提供已编译好的jar包，需要自己构建代码，因此我就再次下载了这个代码源码。当然，由于网络的原因，这个项目的源码也被我导入到了`gitee`上，地址为[gitee.com/funcy/rocke…](https://link.juejin.cn?target=https%3A%2F%2Fgitee.com%2Ffuncy%2Frocketmq-externals.git "https://gitee.com/funcy/rocketmq-externals.git").

这个项目的代码我们并不分析，因此直接在`master`分支上操作即可，

管理后台项目为`rocketmq-console`，主类为`org.apache.rocketmq.console.App`：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70d88f026bcd406fb9a5f23306599854~tplv-k3u1fbpfcp-watermark.awebp)

在启动前，我们需要修改下`application.properties`的配置，找到`rocketmq.config.namesrvAddr`配置，添加`nameServer`的ip与端口，这里我们连接的是本地应用，直接填写`localhost:9876`：

```properties
...
rocketmq.config.namesrvAddr=localhost:9876
...
复制代码
```

启动，结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2bc7934f207f42ff9d63be1789049978~tplv-k3u1fbpfcp-watermark.awebp)

访问`http://localhost:8080`，结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac4f0f12ea614289b8e66f57d8338b54~tplv-k3u1fbpfcp-watermark.awebp)

可以看到`broker`已经出现在`cluster`列表中了，这就表明启动成功了。

### 4. 收发消息测试

`rocketMq`项目的`example`模块下有大量的测试示例，我们选择其一进行消息收发测试。

#### 4.1 启动`Consumer`

我们先找到`org.apache.rocketmq.example.simple.PushConsumer`，代码如下：

```java
public class PushConsumer {

    public static void main(String[] args)             throws InterruptedException, MQClientException {
        String nameServer = "localhost:9876";
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("CID_JODIE_1");
        consumer.setNamesrvAddr(nameServer);
        consumer.subscribe("TopicTest", "*");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        //wrong time format 2017_0422_221800
        consumer.setConsumeTimestamp("20181109221800");
        consumer.registerMessageListener(new MessageListenerConcurrently() {

            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,                     ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", 
                    Thread.currentThread().getName(), msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.printf("Consumer Started.%n");
    }
}
复制代码
```

这个`Consumer`监听的`topic`是`TopicTest`，后面我们就会往这个`topic`发送消息。另外，需要注意`nameServer`的配置，我们是在本地启动的`nameServer`，因此这里配置的是`localhost:9876`。

运行`main()`方法，结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6de3c3f6723a4fd4aa43407ff68bf3d2~tplv-k3u1fbpfcp-watermark.awebp)

#### 4.2 启动`Producer`

我们找到 `org.apache.rocketmq.example.simple.Producer` 类，代码如下：

```java
public class Producer {

    public static void main(String[] args)             throws MQClientException, InterruptedException {
        String nameServer = "localhost:9876";
        DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
        producer.setNamesrvAddr(nameServer);
        producer.start();

        for (int i = 0; i < 10; i++)
            try {
                {
                    Message msg = new Message("TopicTest",
                        "TagA",
                        "OrderID188",
                        "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                    SendResult sendResult = producer.send(msg);
                    System.out.printf("%s%n", sendResult);
                }

            } catch (Exception e) {
                e.printStackTrace();
            }

        producer.shutdown();
    }
}
复制代码
```

同样地，这里使用的是的`nameServer`地址是`localhost:9876`，`topic` 是`TopicTest`，运行，结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28988075e4804f86bc195c168b0b376d~tplv-k3u1fbpfcp-watermark.awebp)

再回过头看看`PushConsumer`的控制台：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73f8d1ca9d884645a39b2f70e25f9c93~tplv-k3u1fbpfcp-watermark.awebp)

可以看到，`Producer`发送消息成功了，`PushConsumer`也成功获取到消息了。

#### 4.3 异常分析

如图所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d1b74db64c44773a9538576850c3f86~tplv-k3u1fbpfcp-watermark.awebp)

如果出现异常：

```
org.apache.rocketmq.client.exception.MQClientException: 
No route info of this topic: TopicTest
复制代码
```

这表明当前`broker`中没有`TopicTest`的`topic`，这时我们可以手动创建`topic`，也可以在启动时指定`autoCreateTopicEnable=true`.

如果是按上面步骤进行的，请确认下`org.apache.rocketmq.broker.BrokerStartup`是否配置启动参数

```
-n localhost:9876 autoCreateTopicEnable=true
复制代码
```

配置方式就按`3.3节`的方式配置就行了。

### 5. 总结

本文主要介绍了`rocketMq`的基本架构，通过源码展示了`rocketMq`的启动方式，最后通过`rocketMq`项目下`example`模块中的测试代码展示了消息的收发过程。

总的来说，本文还是在准备源码分析的环境，下篇文章开始，我们就正式开始`rocketMq`的源码分析了。
