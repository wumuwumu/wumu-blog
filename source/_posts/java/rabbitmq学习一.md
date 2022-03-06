---
title: rabbitmq基本学习1
abbrlink: 764d049d
date: 2021-01-04 15:00:00
---

# 安装

> https://packagecloud.io/rabbitmq/

## 安装erlang

```bash
curl -s https://packagecloud.io/install/repositories/rabbitmq/erlang/script.rpm.sh | sudo bash
dnf install erlang
```

## 安装rabbitmq

```
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash
```

## 启动后台管理界面

```bash
rabbitmq-plugins enable rabbitmq_management
```

## 相关端口

Listening ports：3个端口（5672,25672,15672）;

　　5672对应的是amqp，25672对应的是clustering，15672对应的是http（也就是我们登录RabbitMQ后台管理时用的端口）。

　　25672对应的是集群，15672对应的是后台管理。因为RabbitMQ遵循Ampq协议，所以5672对应的就是RabbitMQ的通信了。

# rabbitMQ常用的命令

启动监控管理器：rabbitmq-plugins enable rabbitmq_management
 关闭监控管理器：rabbitmq-plugins disable rabbitmq_management
 启动rabbitmq：rabbitmq-service start
 关闭rabbitmq：rabbitmq-service stop
 查看所有的队列：rabbitmqctl list_queues
 清除所有的队列：rabbitmqctl reset
 关闭应用：rabbitmqctl stop_app
 启动应用：rabbitmqctl start_app

**用户和权限设置**
 添加用户：rabbitmqctl add_user username password
 分配角色：rabbitmqctl set_user_tags username administrator
 新增虚拟主机：rabbitmqctl add_vhost  vhost_name
 将新虚拟主机授权给新用户：`rabbitmqctl set_permissions -p vhost_name username “.*” “.*” “.*”`(后面三个”*”代表用户拥有配置、写、读全部权限)

**角色说明**

- 超级管理员(administrator)
   可登陆管理控制台，可查看所有的信息，并且可以对用户，策略(policy)进行操作。
- 监控者(monitoring)
   可登陆管理控制台，同时可以查看rabbitmq节点的相关信息(进程数，内存使用情况，磁盘使用情况等)
- 策略制定者(policymaker)
   可登陆管理控制台, 同时可以对policy进行管理。但无法查看节点的相关信息(上图红框标识的部分)。
- 普通管理者(management)
   仅可登陆管理控制台，无法看到节点信息，也无法对策略进行管理。
- 其他
   无法登陆管理控制台，通常就是普通的生产者和消费者。

# 创建用户

guest默认不能远程登陆

```bash
#RabbitMQ新增账号密码
rabbitmqctl add_user test 123456
#设置成管理员角色
rabbitmqctl  set_user_tags  test  administrator
#设置权限
rabbitmqctl set_permissions -p "/" test ".*" ".*" ".*"
#查看用户列表
rabbitmqctl list_users
```

# 基本操作

## 队列

### 相关属性

1. queue:声明的队列名称，同一个队列在声明之后不能修改属性。
2. durable：是否持久化，是否将队列持久化到mnesia数据库中，有专门的表保存我们的队列声明。
3.  exclusive：排外，①当前定义的队列是connection的channel是共享的，其他的connection是访问不到的。②当connection关闭的时候，队列将被删除。
4. autoDelete：自动删除，当最后一个consumer（消费者）断开之后，队列将自动删除。

------

5. *arguments*：参数是rabbitmq的一个扩展，功能非常强大，基本是AMPQ中没有的。

- x-message-ttl：Number ，发布的消息在队列中存在多长时间后被取消（单位毫秒） 可以对单个消息设置过期时间
- x-expires：Number

当Queue（队列）在指定的时间未被访问，则队列将被自动删除。

- x-max-length：Number

队列所能容下消息的最大长度。当超出长度后，新消息将会覆盖最前面的消息，类似于Redis的LRU算法。

- x-max-length-bytes：Number

限定队列的最大占用空间，当超出后也使用类似于Redis的LRU算法。

- x-overflow：String

设置队列溢出行为。这决定了当达到队列的最大长度时，消息会发生什么。有效值为Drop Head或Reject Publish。

- x-dead-letter-exchange：String
   如果消息被拒绝或过期或者超出max，将向其重新发布邮件的交换的可选名称
- x-dead-letter-routing-key：String

如果不定义，则默认为溢出队列的routing-key，因此，一般和6一起定义。

- x-max-priority：Number

如果将一个队列加上优先级参数，那么该队列为优先级队列。

1）、给队列加上优先级参数使其成为优先级队列

x-max-priority=10【值不要太大，本质是一个树结构】

2）、给消息加上优先级属性

- x-queue-mode：String

队列类型　　x-queue-mode=lazy　　懒队列，在磁盘上尽可能多地保留消息以减少RAM使用；如果未设置，则队列将保留内存缓存以尽可能快地传递消息。

- x-queue-master-locator：String

将队列设置为主位置模式，确定在节点集群上声明时队列主位置所依据的规则。

### java代码

```java
ConnectionFactory connectionFactory = new ConnectionFactory();
    connectionFactory.setHost("192.168.100.11");
    connectionFactory.setPort(5672);
    connectionFactory.setUsername("test");
    connectionFactory.setPassword("test");
    try(Connection connection = connectionFactory.newConnection();
        Channel channel = connection.createChannel()) {
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        channel.addConfirmListener(new ConfirmListener(){

          @Override
          public void handleAck(long deliveryTag, boolean multiple) throws IOException {
            System.out.println("handleack "+deliveryTag);
          }

          @Override
          public void handleNack(long deliveryTag, boolean multiple) throws IOException {
            System.out.println("handleNack "+deliveryTag);
          }
        });
        
        channel.addReturnListener(returnMessage -> System.out.println("返回的消息 "+returnMessage));
        String message = RandomStringUtils.randomAlphanumeric(10)+ " hello  !!";
      System.out.println("发送的消息  "+ message);
        channel.basicPublish("",QUEUE_NAME,null,message.getBytes(StandardCharsets.UTF_8));
    } catch (TimeoutException e) {
      e.printStackTrace();
    } catch (IOException e) {
      e.printStackTrace();
    }
```

## 消息发送确认机制

### **事务机制**

这里首先探讨下RabbitMQ事务机制。

RabbitMQ中与事务机制有关的方法有三个：txSelect(), txCommit()以及txRollback(), txSelect用于将当前channel设置成transaction模式，txCommit用于提交事务，txRollback用于回滚事务，在通过txSelect开启事务之后，我们便可以发布消息给broker代理服务器了，如果txCommit提交成功了，则消息一定到达了broker了，如果在txCommit执行之前broker异常崩溃或者由于其他原因抛出异常，这个时候我们便可以捕获异常通过txRollback回滚事务了。

关键代码：

```java
channel.txSelect();
channel.basicPublish(ConfirmConfig.exchangeName, ConfirmConfig.routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, ConfirmConfig.msg_10B.getBytes());
channel.txCommit();
```

通过wirkshark抓包（ip.addr==xxx.xxx.xxx.xxx && amqp），可以看到：
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMjE3MTYzMDM5NjY3?x-oss-process=image/format,png)
（注意这里的Tx.Commit与Tx.Commit-Ok之间的时间间隔294ms，由此可见事务还是很耗时的。）

我们先来看看没有事务的通信过程是什么样的：
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMjE3MTYzMDU4MjYx?x-oss-process=image/format,png)
可以看到带事务的多了四个步骤：

- client发送Tx.Select
- broker发送Tx.Select-Ok(之后publish)
- client发送Tx.Commit
- broker发送Tx.Commit-Ok

下面我们来看下事务回滚是什么样子的。关键代码如下：

```java
try {
    channel.txSelect();
    channel.basicPublish(exchange, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, msg.getBytes());
    int result = 1 / 0;
    channel.txCommit();
} catch (Exception e) {
    e.printStackTrace();
    channel.txRollback();
}
```

同样通过wireshark抓包可以看到：
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMjE3MTYzMTEwMDgz?x-oss-process=image/format,png)
代码中先是发送了消息至broker中但是这时候发生了异常，之后在捕获异常的过程中进行事务回滚。

事务确实能够解决producer与broker之间消息确认的问题，只有消息成功被broker接受，事务提交才能成功，否则我们便可以在捕获异常进行事务回滚操作同时进行消息重发，但是使用事务机制的话会降低RabbitMQ的性能，那么有没有更好的方法既能保障producer知道消息已经正确送到，又能基本上不带来性能上的损失呢？从AMQP协议的层面看是没有更好的方法，但是RabbitMQ提供了一个更好的方案，即将channel信道设置成confirm模式。

### **Confirm模式**

#### **概述**

上面我们介绍了RabbitMQ可能会遇到的一个问题，即生成者不知道消息是否真正到达broker，随后通过AMQP协议层面为我们提供了事务机制解决了这个问题，但是采用事务机制实现会降低RabbitMQ的消息吞吐量，那么有没有更加高效的解决方式呢？答案是采用Confirm模式。

#### **producer端confirm模式的实现原理**

生产者将信道设置成confirm模式，一旦信道进入confirm模式，所有在该信道上面发布的消息都会被指派一个唯一的ID(从1开始)，一旦消息被投递到所有匹配的队列之后，broker就会发送一个确认给生产者（包含消息的唯一ID）,这就使得生产者知道消息已经正确到达目的队列了，如果消息和队列是可持久化的，那么确认消息会将消息写入磁盘之后发出，broker回传给生产者的确认消息中deliver-tag域包含了确认消息的序列号，此外broker也可以设置basic.ack的multiple域，表示到这个序列号之前的所有消息都已经得到了处理。

confirm模式最大的好处在于他是异步的，一旦发布一条消息，生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用便可以通过回调方法来处理该确认消息，如果RabbitMQ因为自身内部错误导致消息丢失，就会发送一条nack消息，生产者应用程序同样可以在回调方法中处理该nack消息。

在channel 被设置成 confirm 模式之后，所有被 publish 的后续消息都将被 confirm（即 ack） 或者被nack一次。但是没有对消息被 confirm 的快慢做任何保证，并且同一条消息不会既被 confirm又被nack 。

#### **开启confirm模式的方法**

生产者通过调用channel的confirmSelect方法将channel设置为confirm模式，如果没有设置no-wait标志的话，broker会返回confirm.select-ok表示同意发送者将当前channel信道设置为confirm模式(从目前RabbitMQ最新版本3.6来看，如果调用了channel.confirmSelect方法，默认情况下是直接将no-wait设置成false的，也就是默认情况下broker是必须回传confirm.select-ok的)。
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMjE3MTYzMTMzMTgz?x-oss-process=image/format,png)

> 已经在transaction事务模式的channel是不能再设置成confirm模式的，即这两种模式是不能共存的。

#### **编程模式**

对于固定消息体大小和线程数，如果消息持久化，生产者confirm(或者采用事务机制)，消费者ack那么对性能有很大的影响.

消息持久化的优化没有太好方法，用更好的物理存储（SAS, SSD, RAID卡）总会带来改善。生产者confirm这一环节的优化则主要在于客户端程序的优化之上。归纳起来，客户端实现生产者confirm有三种编程方式：

1. 普通confirm模式：每发送一条消息后，调用waitForConfirms()方法，等待服务器端confirm。实际上是一种串行confirm了。
2. 批量confirm模式：每发送一批消息后，调用waitForConfirms()方法，等待服务器端confirm。
3. 异步confirm模式：提供一个回调方法，服务端confirm了一条或者多条消息后Client端会回调这个方法。

从编程实现的复杂度上来看：
**第1种**
普通confirm模式最简单，publish一条消息后，等待服务器端confirm,如果服务端返回false或者超时时间内未返回，客户端进行消息重传。
关键代码如下：

```java
channel.basicPublish(ConfirmConfig.exchangeName, ConfirmConfig.routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, ConfirmConfig.msg_10B.getBytes());
if(!channel.waitForConfirms()){
	System.out.println("send message failed.");
}
```

wirkShark抓包可以看到如下：
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMjE3MTYzMjA0Mjg4?x-oss-process=image/format,png)
(注意这里的Publish与Ack的时间间隔：305ms 4ms 4ms 15ms 5ms… )

**第二种**
批量confirm模式稍微复杂一点，客户端程序需要定期（每隔多少秒）或者定量（达到多少条）或者两则结合起来publish消息，然后等待服务器端confirm, 相比普通confirm模式，批量极大提升confirm效率，但是问题在于一旦出现confirm返回false或者超时的情况时，客户端需要将这一批次的消息全部重发，这会带来明显的重复消息数量，并且，当消息经常丢失时，批量confirm性能应该是不升反降的。
关键代码：

```java
channel.confirmSelect();
for(int i=0;i<batchCount;i++){
	channel.basicPublish(ConfirmConfig.exchangeName, ConfirmConfig.routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, ConfirmConfig.msg_10B.getBytes());
}
if(!channel.waitForConfirms()){
	System.out.println("send message failed.");
}
```

**第三种**
异步confirm模式的编程实现最复杂，Channel对象提供的ConfirmListener()回调方法只包含deliveryTag（当前Chanel发出的消息序号），我们需要自己为每一个Channel维护一个unconfirm的消息序号集合，每publish一条数据，集合中元素加1，每回调一次handleAck方法，unconfirm集合删掉相应的一条（multiple=false）或多条（multiple=true）记录。从程序运行效率上看，这个unconfirm集合最好采用有序集合SortedSet存储结构。实际上，SDK中的waitForConfirms()方法也是通过SortedSet维护消息序号的。
关键代码：

```java
 SortedSet<Long> confirmSet = Collections.synchronizedSortedSet(new TreeSet<Long>());
 channel.confirmSelect();
        channel.addConfirmListener(new ConfirmListener() {
            public void handleAck(long deliveryTag, boolean multiple) throws IOException {
                if (multiple) {
                    confirmSet.headSet(deliveryTag + 1).clear();
                } else {
                    confirmSet.remove(deliveryTag);
                }
            }
            public void handleNack(long deliveryTag, boolean multiple) throws IOException {
            	System.out.println("Nack, SeqNo: " + deliveryTag + ", multiple: " + multiple);
                if (multiple) {
                    confirmSet.headSet(deliveryTag + 1).clear();
                } else {
                    confirmSet.remove(deliveryTag);
                }
            }
        });

        while (true) {
            long nextSeqNo = channel.getNextPublishSeqNo();
            channel.basicPublish(ConfirmConfig.exchangeName, ConfirmConfig.routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, ConfirmConfig.msg_10B.getBytes());
            confirmSet.add(nextSeqNo);
        }
```

SDK中waitForConfirms方法实现：

```java
/** Set of currently unconfirmed messages (i.e. messages that have
 *  not been ack'd or nack'd by the server yet. */
private final SortedSet<Long> unconfirmedSet =
        Collections.synchronizedSortedSet(new TreeSet<Long>());

public boolean waitForConfirms(long timeout)
        throws InterruptedException, TimeoutException {
    if (nextPublishSeqNo == 0L)
        throw new IllegalStateException("Confirms not selected");
    long startTime = System.currentTimeMillis();
    synchronized (unconfirmedSet) {
        while (true) {
            if (getCloseReason() != null) {
                throw Utility.fixStackTrace(getCloseReason());
            }
            if (unconfirmedSet.isEmpty()) {
                boolean aux = onlyAcksReceived;
                onlyAcksReceived = true;
                return aux;
            }
            if (timeout == 0L) {
                unconfirmedSet.wait();
            } else {
                long elapsed = System.currentTimeMillis() - startTime;
                if (timeout > elapsed) {
                    unconfirmedSet.wait(timeout - elapsed);
                } else {
                    throw new TimeoutException();
                }
            }
        }
    }
}
```

#### **性能测试**

Client端机器和RabbitMQ机器配置：CPU:24核，2600MHZ, 64G内存，1TB硬盘。
Client端发送消息体大小10B，线程数为1即单线程，消息都持久化处理（deliveryMode:2）。
分别采用事务模式、普通confirm模式，批量confirm模式和异步confirm模式进行producer实验，比对各个模式下的发送性能。
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMjE3MTYzMjI4MDU5?x-oss-process=image/format,png)

发送平均速率：

- 事务模式（tx）：1637.484
- 普通confirm模式(common)：1936.032
- 批量confirm模式(batch)：10432.45
- 异步confirm模式(async)：10542.06

可以看到事务模式性能是最差的，普通confirm模式性能比事务模式稍微好点，但是和批量confirm模式还有异步confirm模式相比，还是小巫见大巫。批量confirm模式的问题在于confirm之后返回false之后进行重发这样会使性能降低，异步confirm模式(async)编程模型较为复杂，至于采用哪种方式，那是仁者见仁智者见智了。

### **消息确认（Consumer端）**

为了保证消息从队列可靠地到达消费者，RabbitMQ提供消息确认机制(message acknowledgment)。消费者在声明队列时，可以指定noAck参数，当noAck=false时，RabbitMQ会等待消费者显式发回ack信号后才从内存(和磁盘，如果是持久化消息的话)中移去消息。否则，RabbitMQ会在队列中消息被消费后立即删除它。

采用消息确认机制后，只要令noAck=false，消费者就有足够的时间处理消息(任务)，不用担心处理消息过程中消费者进程挂掉后消息丢失的问题，因为RabbitMQ会一直持有消息直到消费者显式调用basicAck为止。

当noAck=false时，对于RabbitMQ服务器端而言，队列中的消息分成了两部分：一部分是等待投递给消费者的消息；一部分是已经投递给消费者，但是还没有收到消费者ack信号的消息。如果服务器端一直没有收到消费者的ack信号，并且消费此消息的消费者已经断开连接，则服务器端会安排该消息重新进入队列，等待投递给下一个消费者（也可能还是原来的那个消费者）。

RabbitMQ不会为未ack的消息设置超时时间，它判断此消息是否需要重新投递给消费者的唯一依据是消费该消息的消费者连接是否已经断开。这么设计的原因是RabbitMQ允许消费者消费一条消息的时间可以很久很久。

RabbitMQ管理平台界面上可以看到当前队列中Ready状态和Unacknowledged状态的消息数，分别对应上文中的等待投递给消费者的消息数和已经投递给消费者但是未收到ack信号的消息数。也可以通过命令行来查看上述信息：
![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwMjE3MTYzMjQ1ODQw?x-oss-process=image/format,png)

代码示例（关闭自动消息确认，进行手动ack）：

```java
        QueueingConsumer consumer = new QueueingConsumer(channel);
        channel.basicConsume(ConfirmConfig.queueName, false, consumer);
        
        while(true){
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String msg = new String(delivery.getBody());
     // do something with msg. 
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        }
```

> broker将在下面的情况中对消息进行confirm：
>
> > broker发现当前消息无法被路由到指定的queues中（如果设置了mandatory属性，则broker会发送basic.return）
> > 非持久属性的消息到达了其所应该到达的所有queue中（和镜像queue中）
> > 持久消息到达了其所应该到达的所有queue中（和镜像中），并被持久化到了磁盘（fsync）
> > 持久消息从其所在的所有queue中被consume了（如果必要则会被ack）

basicRecover：是路由不成功的消息可以使用recovery重新发送到队列中。
basicReject：是接收端告诉服务器这个消息我拒绝接收,不处理,可以设置是否放回到队列中还是丢掉，而且只能一次拒绝一个消息,官网中有明确说明不能批量拒绝消息，为解决批量拒绝消息才有了basicNack。
basicNack：可以一次拒绝N条消息，客户端可以设置basicNack方法的multiple参数为true，服务器会拒绝指定了delivery_tag的所有未确认的消息(tag是一个64位的long值，最大值是9223372036854775807)。

## 交换机

有4种不同的交换机类型：

- 直连交换机：Direct exchange
- 扇形交换机：Fanout exchange
- 主题交换机：Topic exchange
- 首部交换机：Headers exchange

# 死信队列

## 死信队列是什么

死信，在官网中对应的单词为“Dead Letter”，可以看出翻译确实非常的简单粗暴。那么死信是个什么东西呢？

“死信”是RabbitMQ中的一种消息机制，当你在消费消息时，如果队列里的消息出现以下情况：

1. 消息被否定确认，使用 `channel.basicNack` 或 `channel.basicReject` ，并且此时`requeue` 属性被设置为`false`。
2. 消息在队列的存活时间超过设置的TTL时间。
3. 消息队列的消息数量已经超过最大队列长度。

那么该消息将成为“死信”。

“死信”消息会被RabbitMQ进行特殊处理，如果配置了死信队列信息，那么该消息将会被丢进死信队列中，如果没有配置，则该消息将会被丢弃。



# 参考

> https://www.jianshu.com/p/469f4608ce5d
>
> https://blog.csdn.net/u013256816/article/details/55515234



