---
title: java创建自定义名称线程工厂
tags:
  - java
abbrlink: '95390e08'
date: 2021-01-21 10:31:35
---

### `ThreadFactoryBuilder`

Google guava 工具类 提供的 `ThreadFactoryBuilder` ,使用链式方法创建。

```java
ThreadFactory guavaThreadFactory = new ThreadFactoryBuilder().setNameFormat("retryClient-pool-").build();
 ExecutorService exec = new ThreadPoolExecutor(1, 1,
         0L, TimeUnit.MILLISECONDS,
         new LinkedBlockingQueue<Runnable>(10),guavaThreadFactory );

 exec.submit(() -> {
     logger.info("--记忆中的颜色是什么颜色---");
 });
```

# 自定义`ThreadFactory`

```java
class MssThreadFactory implements ThreadFactory {
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    MssThreadFactory(String namePrefix) {
        this.namePrefix = namePrefix+"-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread( r,namePrefix + threadNumber.getAndIncrement());
        if (t.isDaemon())
            t.setDaemon(true);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

# `BasicThreadFactory`

Apache commons-lang3 提供的 `BasicThreadFactory`.

```java
ThreadFactory basicThreadFactory = new BasicThreadFactory.Builder()
        .namingPattern("basicThreadFactory-").build();

ExecutorService exec = new ThreadPoolExecutor(1, 1,
        0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>(10),basicThreadFactory );
exec.submit(() -> {
    logger.info("--记忆中的颜色是什么颜色---");
});
```

