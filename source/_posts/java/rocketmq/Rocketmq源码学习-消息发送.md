---
title: Rocketmq源码学习-消息发送
tags:
  - java
  - rocketmq
abbrlink: a765c6f3
date: 2022-03-04 19:00:00
---

本篇文章主要描述rocketmq消息发送过程的一些逻辑和实现的方法

# 概念

## 发送组

（1）生产者组（Producer Group）

同一类 Producer 的集合，这类 Producer 发送同一类消息且发送逻辑一致。如果发送的是事物消息且原始生产者在发送之后崩溃，则 Broker 服务器会联系同一生产者组的其他生产者实例以提交或回溯消费。

（2）生产者实例（Producer Instance）

一个生产者组中部署了很多个进程，每一个进程都称为一个生产者实例。

## 发送流程

![这里写图片描述](https://img-blog.csdn.net/20180810120551126?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NTg3MzMz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

# 发送例子

- 消息发送者步骤

（1）创建消息生产者 producer，并指定生产者组名

（2）指定 Nameserver 地址

（3）启动 producer

（4）创建消息对象，指定主题 Topic、Tag 和消息体

（5）发送消息

（6）关闭生产者 producer

```java
/*
<--引入依赖-->
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.4.0</version>
</dependency>
*/

//发送同步消息
public class SyncProducer {
    public static void main(String[] args) throws Exception {
        // 实例化消息生产者Producer
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        // 设置NameServer的地址
        producer.setNamesrvAddr("localhost:9876");
        // 启动Producer实例
        producer.start();
        for (int i = 0; i < 100; i++) {
            // 创建消息，并指定Topic，Tag和消息体
            Message msg = new Message("TopicTest" /* Topic */,
            "TagA" /* Tag */,
            ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
            );
            // 发送消息到一个Broker
            SendResult sendResult = producer.send(msg);
            // 通过sendResult返回消息是否成功送达
            System.out.printf("%s%n", sendResult);
        }
        // 如果不再发送消息，关闭Producer实例。
        producer.shutdown();
    }
}

//发送异步消息
public class AsyncProducer {
    public static void main(String[] args) throws Exception {
        // 实例化消息生产者Producer
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        // 设置NameServer的地址
        producer.setNamesrvAddr("localhost:9876");
        // 启动Producer实例
        producer.start();
        producer.setRetryTimesWhenSendAsyncFailed(0);
        for (int i = 0; i < 100; i++) {
                final int index = i;
                // 创建消息，并指定Topic，Tag和消息体
                Message msg = new Message("TopicTest",
                    "TagA",
                    "OrderID188",
                    "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                // SendCallback接收异步返回结果的回调
                producer.send(msg, new SendCallback() {
                    @Override
                    public void onSuccess(SendResult sendResult) {
                        System.out.printf("%-10d OK %s %n", index,
                            sendResult.getMsgId());
                    }
                    @Override
                    public void onException(Throwable e) {
                      System.out.printf("%-10d Exception %s %n", index, e);
                      e.printStackTrace();
                    }
                });
        }
        // 如果不再发送消息，关闭Producer实例。
        producer.shutdown();
    }
}
```

# 时序图

![说明：发送同步消息，DefaultMQProducer#send(Message) 对 DefaultMQProducerImpl#send(Message) 进行封装。](https://img-blog.csdn.net/20180810120616411?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NTg3MzMz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

# 客户端发送

## DefaultMQProducer#send(Message)

```java
@Override
public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
     return this.defaultMQProducerImpl.send(msg);
}
```

## DefaultMQProducerImpl#sendDefaultImpl()

```java
public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return send(msg, this.defaultMQProducer.getSendMsgTimeout());
}

public SendResult send(Message msg, long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.sendDefaultImpl(msg, CommunicationMode.SYNC, null, timeout);
}

private SendResult sendDefaultImpl(//
    Message msg, //
    final CommunicationMode communicationMode, //
    final SendCallback sendCallback, //
    final long timeout//
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    // 校验 Producer 处于运行状态
    this.makeSureStateOK();
    // 校验消息格式
    Validators.checkMessage(msg, this.defaultMQProducer);
    //
    final long invokeID = random.nextLong(); // 调用编号；用于下面打印日志，标记为同一次发送消息
    long beginTimestampFirst = System.currentTimeMillis();
    long beginTimestampPrev = beginTimestampFirst;
    long endTimestamp = beginTimestampFirst;
    // 获取 Topic路由信息
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        MessageQueue mq = null; // 最后选择消息要发送到的队列
        Exception exception = null;
        SendResult sendResult = null; // 最后一次发送结果
        int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1; // 同步多次调用
        int times = 0; // 第几次发送
        String[] brokersSent = new String[timesTotal]; // 存储每次发送消息选择的broker名
        // 循环调用发送消息，直到成功
        for (; times < timesTotal; times++) {
            String lastBrokerName = null == mq ? null : mq.getBrokerName();
            MessageQueue tmpmq = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName); // 选择消息要发送到的队列
            if (tmpmq != null) {
                mq = tmpmq;
                brokersSent[times] = mq.getBrokerName();
                try {
                    beginTimestampPrev = System.currentTimeMillis();
                    // 调用发送消息核心方法
                    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);
                    endTimestamp = System.currentTimeMillis();
                    // 更新Broker可用性信息
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                    switch (communicationMode) {
                        case ASYNC:
                            return null;
                        case ONEWAY:
                            return null;
                        case SYNC:
                            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) { // 同步发送成功但存储有问题时 && 配置存储异常时重新发送开关 时，进行重试
                                    continue;
                                }
                            }
                            return sendResult;
                        default:
                            break;
                    }
                } catch (RemotingException e) { // 打印异常，更新Broker可用性信息，更新继续循环
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                    log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                    log.warn(msg.toString());
                    exception = e;
                    continue;
                } catch (MQClientException e) { // 打印异常，更新Broker可用性信息，继续循环
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                    log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                    log.warn(msg.toString());
                    exception = e;
                    continue;
                } catch (MQBrokerException e) { // 打印异常，更新Broker可用性信息，部分情况下的异常，直接返回，结束循环
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                    log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                    log.warn(msg.toString());
                    exception = e;
                    switch (e.getResponseCode()) {
                        // 如下异常continue，进行发送消息重试
                        case ResponseCode.TOPIC_NOT_EXIST:
                        case ResponseCode.SERVICE_NOT_AVAILABLE:
                        case ResponseCode.SYSTEM_ERROR:
                        case ResponseCode.NO_PERMISSION:
                        case ResponseCode.NO_BUYER_ID:
                        case ResponseCode.NOT_IN_CURRENT_UNIT:
                            continue;
                        // 如果有发送结果，进行返回，否则，抛出异常；
                        default:
                            if (sendResult != null) {
                                return sendResult;
                            }
                            throw e;
                    }
                } catch (InterruptedException e) {
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                    log.warn(String.format("sendKernelImpl exception, throw exception, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                    log.warn(msg.toString());
                    throw e;
                }
            } else {
                break;
            }
        }
        // 返回发送结果
        if (sendResult != null) {
            return sendResult;
        }
        // 根据不同情况，抛出不同的异常
        String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s", times, System.currentTimeMillis() - beginTimestampFirst,
                msg.getTopic(), Arrays.toString(brokersSent)) + FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);
        MQClientException mqClientException = new MQClientException(info, exception);
        if (exception instanceof MQBrokerException) {
            mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());
        } else if (exception instanceof RemotingConnectException) {
            mqClientException.setResponseCode(ClientErrorCode.CONNECT_BROKER_EXCEPTION);
        } else if (exception instanceof RemotingTimeoutException) {
            mqClientException.setResponseCode(ClientErrorCode.ACCESS_BROKER_TIMEOUT);
        } else if (exception instanceof MQClientException) {
            mqClientException.setResponseCode(ClientErrorCode.BROKER_NOT_EXIST_EXCEPTION);
        }
        throw mqClientException;
    }
    // Namesrv找不到异常
    List<String> nsList = this.getmQClientFactory().getMQClientAPIImpl().getNameServerAddressList();
    if (null == nsList || nsList.isEmpty()) {
        throw new MQClientException(
            "No name server address, please set it." + FAQUrl.suggestTodo(FAQUrl.NAME_SERVER_ADDR_NOT_EXIST_URL), null).setResponseCode(ClientErrorCode.NO_NAME_SERVER_EXCEPTION);
    }
    // 消息路由找不到异常
    throw new MQClientException("No route info of this topic, " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
        null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);
}
```

- 说明 ：发送消息。步骤：获取消息路由信息，选择要发送到的消息队列，执行消息发送核心方法，并对发送结果进行封装返回。

- 第 1 至 7 行：对`sendsendDefaultImpl(...)`进行封装。

- 第 20 行 ：`invokeID`仅仅用于打印日志，无实际的业务用途。

- 第 25 行 ：获取 Topic路由信息， 详细解析见：DefaultMQProducerImpl#tryToFindTopicPublishInfo()

- 第 30 & 34 行 ：计算调用发送消息到成功为止的最大次数，并进行循环。同步或异步发送消息会调用多次，默认配置为3次。

- 第 36 行 ：选择消息要发送到的队列，详细解析见：MQFaultStrategy

- 第 43 行 ：调用发送消息核心方法，详细解析见：DefaultMQProducerImpl#sendKernelImpl()

- 第 46 行 ：更新`Broker`可用性信息。在选择发送到的消息队列时，会参考`Broker`发送消息的延迟，详细解析见：MQFaultStrategy

- 第 62 至 68 行：当抛出`RemotingException`时，如果进行消息发送失败重试，则可能导致消息发送重复。例如，发送消息超时(`RemotingTimeoutException`)，实际`Broker`接收到该消息并处理成功。因此，`Consumer`在消费时，需要保证幂等性。

## DefaultMQProducerImpl#tryToFindTopicPublishInfo()

```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    // 缓存中获取 Topic发布信息
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    // 当无可用的 Topic发布信息时，从Namesrv获取一次
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }
    // 若获取的 Topic发布信息时候可用，则返回
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else { // 使用 {@link DefaultMQProducer#createTopicKey} 对应的 Topic发布信息。用于 Topic发布信息不存在 && Broker支持自动创建Topic
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

- 说明 ：获得 Topic发布信息。优先从缓存`topicPublishInfoTable`，其次从`Namesrv`中获得。
- 第 3 行 ：从缓存`topicPublishInfoTable`中获得 Topic发布信息。
- 第 5 至 9 行 ：从 `Namesrv` 中获得 Topic发布信息。
- 第 13 至 17 行 ：当从 `Namesrv` 无法获取时，使用 `{@link DefaultMQProducer#createTopicKey}` 对应的 Topic发布信息。目的是当 Broker 开启自动创建 Topic开关时，Broker 接收到消息后自动创建Topic。

        一个topic分布在多个Broker，一个Broker包含多个Queue(brokerName、读队列个数、写队列个数、权限、同步或异步)；从缓存中获取topic路由信息；没有则从namesrv获取；没有使用默认topic获取路由配置信息。

        从NameServer获取配置信息，使用ReentrantLock，设置超时3s；基于Netty从namesrv获取配置信息；然后更新topic本地缓存，需要同步更新发送者和消费者的topic缓存

## 选择合适的队列

1. 是否开启消息失败延迟规避机制

2. 本地变量ThreadLocal 保存上一次发送的消息队列下标，消息发送使用轮询机制获取下一个发送消息队列。同时topic发送有异常延迟，确保选中的消息队列所在broker正常

3. 当前消息队列是否可用

发送消息延迟机制；MQFaultStrategy（latencyMax最大延迟时间 end-start为消息延迟时间，如果失败 则将这个broker的isolation为true，同时这个broker在5分钟内不提供服务等待回复）

```dart
if (this.sendLatencyFaultEnable) {
    try {
       int index = tpInfo.getSendWhichQueue().getAndIncrement();
       for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
           int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
           if (pos < 0)  pos = 0;
           MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
           if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {   // 判断broker是否被规避
               if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                   return mq;
           }
       }
       final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();  //选择
       int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
       if (writeQueueNums > 0) {
       final MessageQueue mq = tpInfo.selectOneMessageQueue();
       if (notBestBroker != null) {
           mq.setBrokerName(notBestBroker);
           mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
       }
       return mq;  
   }          
}
return tpInfo.selectOneMessageQueue(lastBrokerName);
```

4、故障延迟机制FaultItem

开启故障延迟则会构造FaultItem记录，在某一时刻前都当做故障(brokeName、发送消息异常时间点、这个时间点都为故障)

4.1.首先选择一个broker==lastBrokerName并且可用的一个队列（也就是该队列并没有因为延迟过长而被加进了延迟容错对象latencyFaultTolerance 中）

4.2.如果第一步中没有找到合适的队列，此时舍弃broker==lastBrokerName这个条件，选择一个相对较好的broker来发送

4.3. 选择一个队列来发送，一般都是取模方式来获取

也就是当Producer发送消息时间过长，逻辑上N秒内Broker不可用。例如发送时间超过15000ms，broker则在60000ms内不可用

```java
private final ConcurrentHashMap<String, FaultItem> faultItemTable = new ConcurrentHashMap<String, FaultItem>(16);
private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};
class FaultItem implements Comparable<FaultItem> {
    private final String name;
    private volatile long currentLatency;
    private volatile long startTimestamp;
}
public void updateFaultItem(final String name, final long currentLatency, final long notAvailableDuration) {
    FaultItem old = this.faultItemTable.get(name);
    if (null == old) {
        final FaultItem faultItem = new FaultItem(name);
        faultItem.setCurrentLatency(currentLatency);
        faultItem.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);

        old = this.faultItemTable.putIfAbsent(name, faultItem);
        if (old != null) {
            old.setCurrentLatency(currentLatency);
            old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
        }
    }
}
```

## MQFaultStrategy详解

![这里写图片描述](https://img-blog.csdn.net/20180810124033459?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NTg3MzMz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```java
 public class MQFaultStrategy {
    private final static Logger log = ClientLogger.getLog();

    /**
      * 延迟故障容错，维护每个Broker的发送消息的延迟
      * key：brokerName
      */
    private final LatencyFaultTolerance<String> latencyFaultTolerance = new LatencyFaultToleranceImpl();
    /**
      * 发送消息延迟容错开关
      */
    private boolean sendLatencyFaultEnable = false;
    /**
      * 延迟级别数组
      */
    private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
    /**
      * 不可用时长数组
     */
    private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};

    /**
      * 根据 Topic发布信息 选择一个消息队列
      *
      * @param tpInfo Topic发布信息
      * @param lastBrokerName brokerName
      * @return 消息队列
      */
    public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
        if (this.sendLatencyFaultEnable) {
            try {
                // 获取 brokerName=lastBrokerName && 可用的一个消息队列
                int index = tpInfo.getSendWhichQueue().getAndIncrement();
                for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                    int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                    if (pos < 0)
                        pos = 0;
                    MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                    if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                        if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                            return mq;
                    }
                }
                // 选择一个相对好的broker，并获得其对应的一个消息队列，不考虑该队列的可用性
                final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
                int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
                if (writeQueueNums > 0) {
                    final MessageQueue mq = tpInfo.selectOneMessageQueue();
                    if (notBestBroker != null) {
                        mq.setBrokerName(notBestBroker);
                        mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                    }
                    return mq;
                } else {
                    latencyFaultTolerance.remove(notBestBroker);
                }
            } catch (Exception e) {
                log.error("Error occurred when selecting message queue", e);
            }
            // 选择一个消息队列，不考虑队列的可用性
            return tpInfo.selectOneMessageQueue();
        }
        // 获得 lastBrokerName 对应的一个消息队列，不考虑该队列的可用性
        return tpInfo.selectOneMessageQueue(lastBrokerName);
    }

    /**
      * 更新延迟容错信息
      *
      * @param brokerName brokerName
      * @param currentLatency 延迟
      * @param isolation 是否隔离。当开启隔离时，默认延迟为30000。目前主要用于发送消息异常时
      */
    public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
        if (this.sendLatencyFaultEnable) {
            long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
            this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
        }
    }

    /**
      * 计算延迟对应的不可用时间
      *
      * @param currentLatency 延迟
      * @return 不可用时间
      */
    private long computeNotAvailableDuration(final long currentLatency) {
        for (int i = latencyMax.length - 1; i >= 0; i--) {
            if (currentLatency >= latencyMax[i])
                return this.notAvailableDuration[i];
        }
        return 0;
    }
```

- 说明 ：`Producer`消息发送容错策略。默认情况下容错策略关闭，即`sendLatencyFaultEnable=false`。
- 第 30 至 62 行 ：容错策略选择消息队列逻辑。优先获取可用队列，其次选择一个broker获取队列，最差返回任意broker的一个队列。
- 第 64 行 ：未开启容错策略选择消息队列逻辑。
- 第 74 至 79 行 ：更新延迟容错信息。当 `Producer` 发送消息时间过长，则逻辑认为N秒内不可用。按照`latencyMax`，`notAvailableDuration`的配置，对应如下：
  
  | Producer发送消息消耗时长 | Broker不可用时长  |
  | ---------------- | ------------ |
  | >= 15000 ms      | 600 1000 ms  |
  | >= 3000 ms       | 180 1000 ms  |
  | >= 2000 ms       | 120 1000 ms  |
  | >= 1000 ms       | 60 1000 ms   |
  | >= 550 ms        | 30 * 1000 ms |
  | >= 100 ms        | 0 ms         |
  | >= 50 ms         | 0 ms         |

### LatencyFaultTolerance

```java
public interface LatencyFaultTolerance<T> {

    /**     * 更新对应的延迟和不可用时长     
    *     * @param name 对象     
    * @param currentLatency 延迟     
    * @param notAvailableDuration 不可用时长     
    */
    void updateFaultItem(final T name, final long currentLatency, final long notAvailableDuration);

    /**     
    * 对象是否可用     
    *     
    * @param name 对象     
    * @return 是否可用     
    */
    boolean isAvailable(final T name);

    /**     
    * 移除对象     
    *     
    * @param name 对象     
    */
    void remove(final T name);

    /**     
    * 获取一个对象     
    *     
    * @return 对象     
    */
    T pickOneAtLeast();
}
```

- 说明 ：延迟故障容错接口

### LatencyFaultToleranceImpl

```java
public class LatencyFaultToleranceImpl implements LatencyFaultTolerance<String> {

    /**
     * 对象故障信息Table
     */
    private final ConcurrentHashMap<String, FaultItem> faultItemTable = new ConcurrentHashMap<>(16);
    /**
     * 对象选择Index
     * @see #pickOneAtLeast()
     */
    private final ThreadLocalIndex whichItemWorst = new ThreadLocalIndex();

    @Override
    public void updateFaultItem(final String name, final long currentLatency, final long notAvailableDuration) {
        FaultItem old = this.faultItemTable.get(name);
        if (null == old) {
            // 创建对象
            final FaultItem faultItem = new FaultItem(name);
            faultItem.setCurrentLatency(currentLatency);
            faultItem.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
            // 更新对象
            old = this.faultItemTable.putIfAbsent(name, faultItem);
            if (old != null) {
                old.setCurrentLatency(currentLatency);
                old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
            }
        } else { // 更新对象
            old.setCurrentLatency(currentLatency);
            old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
        }
    }

    @Override
    public boolean isAvailable(final String name) {
        final FaultItem faultItem = this.faultItemTable.get(name);
        if (faultItem != null) {
            return faultItem.isAvailable();
        }
        return true;
    }

    @Override
    public void remove(final String name) {
        this.faultItemTable.remove(name);
    }

    /**
     * 选择一个相对优秀的对象
     *
     * @return 对象
     */
    @Override
    public String pickOneAtLeast() {
        // 创建数组
        final Enumeration<FaultItem> elements = this.faultItemTable.elements();
        List<FaultItem> tmpList = new LinkedList<>();
        while (elements.hasMoreElements()) {
            final FaultItem faultItem = elements.nextElement();
            tmpList.add(faultItem);
        }
        //
        if (!tmpList.isEmpty()) {
            // 打乱 + 排序。TODO 疑问：应该只能二选一。猜测Collections.shuffle(tmpList)去掉。
            Collections.shuffle(tmpList);
            Collections.sort(tmpList);
            // 选择顺序在前一半的对象
            final int half = tmpList.size() / 2;
            if (half <= 0) {
                return tmpList.get(0).getName();
            } else {
                final int i = this.whichItemWorst.getAndIncrement() % half;
                return tmpList.get(i).getName();
            }
        }
        return null;
    }
}
```

- 说明 ：延迟故障容错实现。维护每个对象的信息。

**FaultItem**

```java
class FaultItem implements Comparable<FaultItem> {
    /**     * 对象名     */
    private final String name;
    /**     * 延迟     */
    private volatile long currentLatency;
    /**     * 开始可用时间     */
    private volatile long startTimestamp;

    public FaultItem(final String name) {
        this.name = name;
    }

    /**     * 比较对象     * 可用性 > 延迟 > 开始可用时间     *     * @param other other     * @return 升序     */
    @Override
    public int compareTo(final FaultItem other) {
        if (this.isAvailable() != other.isAvailable()) {
            if (this.isAvailable())
                return -1;

            if (other.isAvailable())
                return 1;
        }

        if (this.currentLatency < other.currentLatency)
            return -1;
        else if (this.currentLatency > other.currentLatency) {
            return 1;
        }

        if (this.startTimestamp < other.startTimestamp)
            return -1;
        else if (this.startTimestamp > other.startTimestamp) {
            return 1;
        }

        return 0;
    }

    /**     * 是否可用：当开始可用时间大于当前时间     *     * @return 是否可用     */
    public boolean isAvailable() {
        return (System.currentTimeMillis() - startTimestamp) >= 0;
    }

    @Override
    public int hashCode() {
        int result = getName() != null ? getName().hashCode() : 0;
        result = 31 * result + (int) (getCurrentLatency() ^ (getCurrentLatency() >>> 32));
        result = 31 * result + (int) (getStartTimestamp() ^ (getStartTimestamp() >>> 32));
        return result;
    }

    @Override
    public boolean equals(final Object o) {
        if (this == o)
            return true;
        if (!(o instanceof FaultItem))
            return false;

        final FaultItem faultItem = (FaultItem) o;

        if (getCurrentLatency() != faultItem.getCurrentLatency())
            return false;
        if (getStartTimestamp() != faultItem.getStartTimestamp())
            return false;
        return getName() != null ? getName().equals(faultItem.getName()) : faultItem.getName() == null;

    }
}
```

## DefaultMQProducerImpl#sendKernelImpl()

     该方法是消息发送核心方法，已经明确往哪个Broker发送消息了，里面设置到消息校验、消息发送前和发送后做的事情(消息轨迹就是在这里处理的后续文章会分析)、构建请求消息体最终调用remotingClient.invoke()并完成netty的网络请求（具体就是创建channel并将数据写入writeAndFlush，通过ChannelFutureListener进行监听返回结果）

```java
private SendResult sendKernelImpl(final Message msg, //
    final MessageQueue mq, //
    final CommunicationMode communicationMode, //
    final SendCallback sendCallback, //
    final TopicPublishInfo topicPublishInfo, //
    final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    // 获取 broker地址
    String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    if (null == brokerAddr) {
        tryToFindTopicPublishInfo(mq.getTopic());
        brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    }
    //
    SendMessageContext context = null;
    if (brokerAddr != null) {
        // 是否使用broker vip通道。broker会开启两个端口对外服务。
        brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);
        byte[] prevBody = msg.getBody(); // 记录消息内容。下面逻辑可能改变消息内容，例如消息压缩。
        try {
            // 设置唯一编号
            MessageClientIDSetter.setUniqID(msg);
            // 消息压缩
            int sysFlag = 0;
            if (this.tryToCompressMessage(msg)) {
                sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
            }
            // 事务
            final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
            if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
                sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
            }
            // hook：发送消息校验
            if (hasCheckForbiddenHook()) {
                CheckForbiddenContext checkForbiddenContext = new CheckForbiddenContext();
                checkForbiddenContext.setNameSrvAddr(this.defaultMQProducer.getNamesrvAddr());
                checkForbiddenContext.setGroup(this.defaultMQProducer.getProducerGroup());
                checkForbiddenContext.setCommunicationMode(communicationMode);
                checkForbiddenContext.setBrokerAddr(brokerAddr);
                checkForbiddenContext.setMessage(msg);
                checkForbiddenContext.setMq(mq);
                checkForbiddenContext.setUnitMode(this.isUnitMode());
                this.executeCheckForbiddenHook(checkForbiddenContext);
            }
            // hook：发送消息前逻辑
            if (this.hasSendMessageHook()) {
                context = new SendMessageContext();
                context.setProducer(this);
                context.setProducerGroup(this.defaultMQProducer.getProducerGroup());
                context.setCommunicationMode(communicationMode);
                context.setBornHost(this.defaultMQProducer.getClientIP());
                context.setBrokerAddr(brokerAddr);
                context.setMessage(msg);
                context.setMq(mq);
                String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
                if (isTrans != null && isTrans.equals("true")) {
                    context.setMsgType(MessageType.Trans_Msg_Half);
                }
                if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
                    context.setMsgType(MessageType.Delay_Msg);
                }
                this.executeSendMessageHookBefore(context);
            }
            // 构建发送消息请求
            SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
            requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
            requestHeader.setTopic(msg.getTopic());
            requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
            requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
            requestHeader.setQueueId(mq.getQueueId());
            requestHeader.setSysFlag(sysFlag);
            requestHeader.setBornTimestamp(System.currentTimeMillis());
            requestHeader.setFlag(msg.getFlag());
            requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
            requestHeader.setReconsumeTimes(0);
            requestHeader.setUnitMode(this.isUnitMode());
            if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) { // 消息重发Topic
                String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
                if (reconsumeTimes != null) {
                    requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
                }
                String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
                if (maxReconsumeTimes != null) {
                    requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
                }
            }
            // 发送消息
            SendResult sendResult = null;
            switch (communicationMode) {
                case ASYNC:
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(//
                        brokerAddr, // 1
                        mq.getBrokerName(), // 2
                        msg, // 3
                        requestHeader, // 4
                        timeout, // 5
                        communicationMode, // 6
                        sendCallback, // 7
                        topicPublishInfo, // 8
                        this.mQClientFactory, // 9
                        this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(), // 10
                        context, //
                        this);
                    break;
                case ONEWAY:
                case SYNC:
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        msg,
                        requestHeader,
                        timeout,
                        communicationMode,
                        context,
                        this);
                    break;
                default:
                    assert false;
                    break;
            }
            // hook：发送消息后逻辑
            if (this.hasSendMessageHook()) {
                context.setSendResult(sendResult);
                this.executeSendMessageHookAfter(context);
            }
            // 返回发送结果
            return sendResult;
        } catch (RemotingException e) {
            if (this.hasSendMessageHook()) {
                context.setException(e);
                this.executeSendMessageHookAfter(context);
            }
            throw e;
        } catch (MQBrokerException e) {
            if (this.hasSendMessageHook()) {
                context.setException(e);
                this.executeSendMessageHookAfter(context);
            }
            throw e;
        } catch (InterruptedException e) {
            if (this.hasSendMessageHook()) {
                context.setException(e);
                this.executeSendMessageHookAfter(context);
            }
            throw e;
        } finally {
            msg.setBody(prevBody);
        }
    }
    // broker为空抛出异常
    throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
}
```

- 说明 ：发送消息核心方法。该方法真正发起网络请求，发送消息给 `Broker`。
- 第 21 行 ：生产消息编号，详细解析见《RocketMQ 源码分析 —— Message 基础》。
- 第 64 至 121 行 ：构建发送消息请求`SendMessageRequestHeader`。
- 第 107 至 117 行 ：执行 `MQClientInstance#sendMessage(...)` 发起网络请求。

# Broker 接收消息

![这里写图片描述](https://img-blog.csdn.net/20180810132243889?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NTg3MzMz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)  
**SendMessageProcessor#sendMessage**

```java
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
    SendMessageContext mqtraceContext;
    switch (request.getCode()) {
        case RequestCode.CONSUMER_SEND_MSG_BACK:
            return this.consumerSendMsgBack(ctx, request);
        default:
            // 解析请求
            SendMessageRequestHeader requestHeader = parseRequestHeader(request);
            if (requestHeader == null) {
                return null;
            }
            // 发送请求Context。在 hook 场景下使用
            mqtraceContext = buildMsgContext(ctx, requestHeader);
            // hook：处理发送消息前逻辑
            this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
            // 处理发送消息逻辑
            final RemotingCommand response = this.sendMessage(ctx, request, mqtraceContext, requestHeader);
            // hook：处理发送消息后逻辑
            this.executeSendMessageHookAfter(response, mqtraceContext);
            return response;
    }
}

private RemotingCommand sendMessage(final ChannelHandlerContext ctx, //
    final RemotingCommand request, //
    final SendMessageContext sendMessageContext, //
    final SendMessageRequestHeader requestHeader) throws RemotingCommandException {

    // 初始化响应
    final RemotingCommand response = RemotingCommand.createResponseCommand(SendMessageResponseHeader.class);
    final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader) response.readCustomHeader();
    response.setOpaque(request.getOpaque());
    response.addExtField(MessageConst.PROPERTY_MSG_REGION, this.brokerController.getBrokerConfig().getRegionId());
    response.addExtField(MessageConst.PROPERTY_TRACE_SWITCH, String.valueOf(this.brokerController.getBrokerConfig().isTraceOn()));

    if (log.isDebugEnabled()) {
        log.debug("receive SendMessage request command, {}", request);
    }

    // 如果未开始接收消息，抛出系统异常
    @SuppressWarnings("SpellCheckingInspection")
    final long startTimstamp = this.brokerController.getBrokerConfig().getStartAcceptSendRequestTimeStamp();
    if (this.brokerController.getMessageStore().now() < startTimstamp) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark(String.format("broker unable to service, until %s", UtilAll.timeMillisToHumanString2(startTimstamp)));
        return response;
    }

    // 消息配置(Topic配置）校验
    response.setCode(-1);
    super.msgCheck(ctx, requestHeader, response);
    if (response.getCode() != -1) {
        return response;
    }

    final byte[] body = request.getBody();

    // 如果队列小于0，从可用队列随机选择
    int queueIdInt = requestHeader.getQueueId();
    TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
    if (queueIdInt < 0) {
        queueIdInt = Math.abs(this.random.nextInt() % 99999999) % topicConfig.getWriteQueueNums();
    }

    //
    int sysFlag = requestHeader.getSysFlag();
    if (TopicFilterType.MULTI_TAG == topicConfig.getTopicFilterType()) {
        sysFlag |= MessageSysFlag.MULTI_TAGS_FLAG;
    }

    // 对RETRY类型的消息处理。如果超过最大消费次数，则topic修改成"%DLQ%" + 分组名，即加入 死信队列(Dead Letter Queue)
    String newTopic = requestHeader.getTopic();
    if (null != newTopic && newTopic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
        // 获取订阅分组配置
        String groupName = newTopic.substring(MixAll.RETRY_GROUP_TOPIC_PREFIX.length());
        SubscriptionGroupConfig subscriptionGroupConfig =
            this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(groupName);
        if (null == subscriptionGroupConfig) {
            response.setCode(ResponseCode.SUBSCRIPTION_GROUP_NOT_EXIST);
            response.setRemark("subscription group not exist, " + groupName + " " + FAQUrl.suggestTodo(FAQUrl.SUBSCRIPTION_GROUP_NOT_EXIST));
            return response;
        }
        // 计算最大可消费次数
        int maxReconsumeTimes = subscriptionGroupConfig.getRetryMaxTimes();
        if (request.getVersion() >= MQVersion.Version.V3_4_9.ordinal()) {
            maxReconsumeTimes = requestHeader.getMaxReconsumeTimes();
        }
        int reconsumeTimes = requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes();
        if (reconsumeTimes >= maxReconsumeTimes) { // 超过最大消费次数
            newTopic = MixAll.getDLQTopic(groupName);
            queueIdInt = Math.abs(this.random.nextInt() % 99999999) % DLQ_NUMS_PER_GROUP;
            topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(newTopic, //
                DLQ_NUMS_PER_GROUP, //
                PermName.PERM_WRITE, 0
            );
            if (null == topicConfig) {
                response.setCode(ResponseCode.SYSTEM_ERROR);
                response.setRemark("topic[" + newTopic + "] not exist");
                return response;
            }
        }
    }

    // 创建MessageExtBrokerInner
    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    msgInner.setTopic(newTopic);
    msgInner.setBody(body);
    msgInner.setFlag(requestHeader.getFlag());
    MessageAccessor.setProperties(msgInner, MessageDecoder.string2messageProperties(requestHeader.getProperties()));
    msgInner.setPropertiesString(requestHeader.getProperties());
    msgInner.setTagsCode(MessageExtBrokerInner.tagsString2tagsCode(topicConfig.getTopicFilterType(), msgInner.getTags()));
    msgInner.setQueueId(queueIdInt);
    msgInner.setSysFlag(sysFlag);
    msgInner.setBornTimestamp(requestHeader.getBornTimestamp());
    msgInner.setBornHost(ctx.channel().remoteAddress());
    msgInner.setStoreHost(this.getStoreHost());
    msgInner.setReconsumeTimes(requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes());

    // 校验是否不允许发送事务消息
    if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()) {
        String traFlag = msgInner.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
        if (traFlag != null) {
            response.setCode(ResponseCode.NO_PERMISSION);
            response.setRemark(
                "the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1() + "] sending transaction message is forbidden");
            return response;
        }
    }

    // 添加消息
    PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
    if (putMessageResult != null) {
        boolean sendOK = false;

        switch (putMessageResult.getPutMessageStatus()) {
            // Success
            case PUT_OK:
                sendOK = true;
                response.setCode(ResponseCode.SUCCESS);
                break;
            case FLUSH_DISK_TIMEOUT:
                response.setCode(ResponseCode.FLUSH_DISK_TIMEOUT);
                sendOK = true;
                break;
            case FLUSH_SLAVE_TIMEOUT:
                response.setCode(ResponseCode.FLUSH_SLAVE_TIMEOUT);
                sendOK = true;
                break;
            case SLAVE_NOT_AVAILABLE:
                response.setCode(ResponseCode.SLAVE_NOT_AVAILABLE);
                sendOK = true;
                break;

            // Failed
            case CREATE_MAPEDFILE_FAILED:
                response.setCode(ResponseCode.SYSTEM_ERROR);
                response.setRemark("create mapped file failed, server is busy or broken.");
                break;
            case MESSAGE_ILLEGAL:
            case PROPERTIES_SIZE_EXCEEDED:
                response.setCode(ResponseCode.MESSAGE_ILLEGAL);
                response.setRemark(
                    "the message is illegal, maybe msg body or properties length not matched. msg body length limit 128k, msg properties length limit 32k.");
                break;
            case SERVICE_NOT_AVAILABLE:
                response.setCode(ResponseCode.SERVICE_NOT_AVAILABLE);
                response.setRemark(
                    "service not available now, maybe disk full, " + diskUtil() + ", maybe your broker machine memory too small.");
                break;
            case OS_PAGECACHE_BUSY:
                response.setCode(ResponseCode.SYSTEM_ERROR);
                response.setRemark("[PC_SYNCHRONIZED]broker busy, start flow control for a while");
                break;
            case UNKNOWN_ERROR:
                response.setCode(ResponseCode.SYSTEM_ERROR);
                response.setRemark("UNKNOWN_ERROR");
                break;
            default:
                response.setCode(ResponseCode.SYSTEM_ERROR);
                response.setRemark("UNKNOWN_ERROR DEFAULT");
                break;
        }

        String owner = request.getExtFields().get(BrokerStatsManager.COMMERCIAL_OWNER);
        if (sendOK) {
            // 统计
            this.brokerController.getBrokerStatsManager().incTopicPutNums(msgInner.getTopic());
            this.brokerController.getBrokerStatsManager().incTopicPutSize(msgInner.getTopic(), putMessageResult.getAppendMessageResult().getWroteBytes());
            this.brokerController.getBrokerStatsManager().incBrokerPutNums();

            // 响应
            response.setRemark(null);
            responseHeader.setMsgId(putMessageResult.getAppendMessageResult().getMsgId());
            responseHeader.setQueueId(queueIdInt);
            responseHeader.setQueueOffset(putMessageResult.getAppendMessageResult().getLogicsOffset());
            doResponse(ctx, request, response);

            // hook：设置发送成功到context
            if (hasSendMessageHook()) {
                sendMessageContext.setMsgId(responseHeader.getMsgId());
                sendMessageContext.setQueueId(responseHeader.getQueueId());
                sendMessageContext.setQueueOffset(responseHeader.getQueueOffset());

                int commercialBaseCount = brokerController.getBrokerConfig().getCommercialBaseCount();
                int wroteSize = putMessageResult.getAppendMessageResult().getWroteBytes();
                int incValue = (int) Math.ceil(wroteSize / BrokerStatsManager.SIZE_PER_COUNT) * commercialBaseCount;

                sendMessageContext.setCommercialSendStats(BrokerStatsManager.StatsType.SEND_SUCCESS);
                sendMessageContext.setCommercialSendTimes(incValue);
                sendMessageContext.setCommercialSendSize(wroteSize);
                sendMessageContext.setCommercialOwner(owner);
            }
            return null;
        } else {
            // hook：设置发送失败到context
            if (hasSendMessageHook()) {
                int wroteSize = request.getBody().length;
                int incValue = (int) Math.ceil(wroteSize / BrokerStatsManager.SIZE_PER_COUNT);

                sendMessageContext.setCommercialSendStats(BrokerStatsManager.StatsType.SEND_FAILURE);
                sendMessageContext.setCommercialSendTimes(incValue);
                sendMessageContext.setCommercialSendSize(wroteSize);
                sendMessageContext.setCommercialOwner(owner);
            }
        }
    } else {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("store putMessage return null");
    }

    return response;
}
```

- `#processRequest()` 说明 ：处理消息请求。
- `#sendMessage()` 说明 ：发送消息，并返回发送消息结果。
- 第 51 至 55 行 ：消息配置(Topic配置）校验，详细解析见：AbstractSendMessageProcessor#msgCheck()。
- 第 60 至 64 行 ：消息队列编号小于0时，Broker 可以设置随机选择一个消息队列。
- 第 72 至 103 行 ：对RETRY类型的消息处理。如果超过最大消费次数，则topic修改成”%DLQ%” + 分组名， 即加 死信队 (Dead Letter Queue)，详细解析见：《RocketMQ 源码分析 —— Topic》。
- 第 105 至 118 行 ：创建`MessageExtBrokerInner`。
- 第 132 ：存储消息，详细解析见：DefaultMessageStore#putMessage()。
- 第 133 至 183 行 ：处理消息发送结果，设置响应结果和提示。
- 第 186 至 214 行 ：发送成功，响应。这里`doResponse(ctx, request, response)`进行响应，最后`return null`，原因是：响应给 `Producer` 可能发生异常，`#doResponse(ctx, request, response)`捕捉了该异常并输出日志。这样做的话，我们进行排查 Broker 接收消息成功后响应是否存在异常会方便很多。

**AbstractSendMessageProcessor#msgCheck**

```java
protected RemotingCommand msgCheck(final ChannelHandlerContext ctx,
                                   final SendMessageRequestHeader requestHeader, final RemotingCommand response) {
    // 检查 broker 是否有写入权限
    if (!PermName.isWriteable(this.brokerController.getBrokerConfig().getBrokerPermission())
        && this.brokerController.getTopicConfigManager().isOrderTopic(requestHeader.getTopic())) {
        response.setCode(ResponseCode.NO_PERMISSION);
        response.setRemark("the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1()
            + "] sending message is forbidden");
        return response;
    }
    // 检查topic是否可以被发送。目前是{@link MixAll.DEFAULT_TOPIC}不被允许发送
    if (!this.brokerController.getTopicConfigManager().isTopicCanSendMessage(requestHeader.getTopic())) {
        String errorMsg = "the topic[" + requestHeader.getTopic() + "] is conflict with system reserved words.";
        log.warn(errorMsg);
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark(errorMsg);
        return response;
    }
    TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
    if (null == topicConfig) { // 不能存在topicConfig，则进行创建
        int topicSysFlag = 0;
        if (requestHeader.isUnitMode()) {
            if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                topicSysFlag = TopicSysFlag.buildSysFlag(false, true);
            } else {
                topicSysFlag = TopicSysFlag.buildSysFlag(true, false);
            }
        }
        // 创建topic配置
        log.warn("the topic {} not exist, producer: {}", requestHeader.getTopic(), ctx.channel().remoteAddress());
        topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageMethod(//
            requestHeader.getTopic(), //
            requestHeader.getDefaultTopic(), //
            RemotingHelper.parseChannelRemoteAddr(ctx.channel()), //
            requestHeader.getDefaultTopicQueueNums(), topicSysFlag);
        if (null == topicConfig) {
            if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                topicConfig =
                    this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(
                        requestHeader.getTopic(), 1, PermName.PERM_WRITE | PermName.PERM_READ,
                        topicSysFlag);
            }
        }
        // 如果没配置
        if (null == topicConfig) {
            response.setCode(ResponseCode.TOPIC_NOT_EXIST);
            response.setRemark("topic[" + requestHeader.getTopic() + "] not exist, apply first please!"
                + FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL));
            return response;
        }
    }
    // 队列编号是否正确
    int queueIdInt = requestHeader.getQueueId();
    int idValid = Math.max(topicConfig.getWriteQueueNums(), topicConfig.getReadQueueNums());
    if (queueIdInt >= idValid) {
        String errorInfo = String.format("request queueId[%d] is illegal, %s Producer: %s",
            queueIdInt,
            topicConfig.toString(),
            RemotingHelper.parseChannelRemoteAddr(ctx.channel()));
        log.warn(errorInfo);
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark(errorInfo);
        return response;
    }
    return response;
}
```

- 说明：校验消息是否正确，主要是Topic配置方面，例如：`Broker` 是否有写入权限，topic配置是否存在，队列编号是否正确。
- 第 11 至 18 行 ：检查Topic是否可以被发送。目前是 `{@link MixAll.DEFAULT_TOPIC}` 不被允许发送。
- 第 20 至 51 行 ：当找不到Topic配置，则进行创建。当然，创建会存在不成功的情况，例如说：`defaultTopic` 的Topic配置不存在，又或者是 存在但是不允许继承，详细解析见《RocketMQ 源码分析 —— Topic》。

**DefaultMessageStore#putMessage**

```java
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
    if (this.shutdown) {
        log.warn("message store has shutdown, so putMessage is forbidden");
        return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
    }

    // 从节点不允许写入
    if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
        long value = this.printTimes.getAndIncrement();
        if ((value % 50000) == 0) {
            log.warn("message store is slave mode, so putMessage is forbidden ");
        }

        return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
    }

    // store是否允许写入
    if (!this.runningFlags.isWriteable()) {
        long value = this.printTimes.getAndIncrement();
        if ((value % 50000) == 0) {
            log.warn("message store is not writeable, so putMessage is forbidden " + this.runningFlags.getFlagBits());
        }

        return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
    } else {
        this.printTimes.set(0);
    }

    // 消息过长
    if (msg.getTopic().length() > Byte.MAX_VALUE) {
        log.warn("putMessage message topic length too long " + msg.getTopic().length());
        return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, null);
    }

    // 消息附加属性过长
    if (msg.getPropertiesString() != null && msg.getPropertiesString().length() > Short.MAX_VALUE) {
        log.warn("putMessage message properties length too long " + msg.getPropertiesString().length());
        return new PutMessageResult(PutMessageStatus.PROPERTIES_SIZE_EXCEEDED, null);
    }

    if (this.isOSPageCacheBusy()) {
        return new PutMessageResult(PutMessageStatus.OS_PAGECACHE_BUSY, null);
    }

    long beginTime = this.getSystemClock().now();
    // 添加消息到commitLog
    PutMessageResult result = this.commitLog.putMessage(msg);

    long eclipseTime = this.getSystemClock().now() - beginTime;
    if (eclipseTime > 500) {
        log.warn("putMessage not in lock eclipse time(ms)={}, bodyLength={}", eclipseTime, msg.getBody().length);
    }
    this.storeStatsService.setPutMessageEntireTimeMax(eclipseTime);

    if (null == result || !result.isOk()) {
        this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
    }

    return result;
}
```

- 说明：存储消息封装，最终存储需要 `CommitLog` 实现。
- 第 7 至 27 行 ：校验 `Broker` 是否可以写入。
- 第 29 至 39 行 ：消息格式与大小校验。
- 第 47 行 ：调用 `CommitLong` 进行存储，详细逻辑见：[《RocketMQ 源码分析 —— Message 存储》](https://blog.csdn.net/m0_37587333/article/details/81561865)

# 参考

[RocketMQ系列3-消息发送流程 - 简书](https://www.jianshu.com/p/bce77e05816c)
