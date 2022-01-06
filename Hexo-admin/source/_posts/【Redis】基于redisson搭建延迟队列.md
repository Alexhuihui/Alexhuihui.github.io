---
title: 【Redis】基于redisson搭建延迟队列
date: 2022-01-06 16:18:56
categories: Redis
tags: [Redis, 延时队列]
---

### 什么是延时队列？

> 它首先具有队列的特性（先进先出），同时还能指定队列中的消息在未来某个时刻被消费。
>
> 应用：抢单成功后，订单1分钟内未支付，自动关闭。

<!-- more --> 

### 实现延时队列

很早之前我有使用过redis的zset数据结构来实现延时队列（自动关闭考试），但是这种方法需要开启定时任务去轮询delayqueue，实时性不好，且无法支持应用的水平扩展。后来有接触到[redisson](https://github.com/redisson/redisson)——一个Java实现的redis客户端，发现用它来实现延时队列可以做的更好。

#### 1、定义接口

```java
public interface Queue {
}
```

#### 2、实现抽象类

首先要实现一个redisson的客户端

```java
@Bean
public RedissonClient getRedissonClient() {
        Config config = new Config();
        SingleServerConfig server = config.useSingleServer();
        String address = StrUtil.format("redis://{}:{}", host, port);
        server.setAddress(address);
        if (StrUtil.isNotEmpty(password)) {
            server.setPassword(password);
        }
        return Redisson.create(config);
}
```

***

抽象类AbstractQueue

```java
public abstract class AbstractQueue implements Queue {

    protected String name;
    protected static Map<String, IQueueListener> listenerMap;
    protected static RedissonClient redissonClient = SpringUtil.getBean(RedissonClient.class);
    protected static String LOCK_PREFIX = "alexmmd:queue:listeners";

    static {
        // put topic and corresponding queueListener into the listenerMap
        listenerMap = new HashMap<>();
        QueueConsumer annotation;
        Map<String, IQueueListener> beansOfType = SpringUtil.getBeansOfType(IQueueListener.class);
        if (MapUtil.isNotEmpty(beansOfType)) {
            for (IQueueListener listener : beansOfType.values()) {
                // bind topic and corresponding listener
                annotation = listener.getClass().getAnnotation(QueueConsumer.class);
                if (ObjectUtil.isNull(annotation)) {
                    continue;
                }
                String topic = annotation.topic();
                if (StrUtil.isNotEmpty(topic)) {
                    listenerMap.put(topic, listener);
                }
                String[] topics = annotation.topics();
                if (ArrayUtil.isEmpty(topics)) {
                    continue;
                }
                for (String s : topics) {
                    listenerMap.put(s, listener);
                }
            }
        }
    }

    public AbstractQueue(String name) {
        this.name = name;
        new Thread(() -> {
            log.info("开启一个线程 {} 监听队列 : {}", Thread.currentThread().getName(), name);
            while (true) {
                try {
                    Pair<String, String> take = this.take();
                    String topic = take.getKey();
                    String body = take.getValue();
                    new Thread(consumer(name, topic, body)).start();
                } catch (Exception e) {
                    log.error("监听队列线程错误，{}", e.getMessage());
                }
            }
        }).start();
    }

    private Runnable consumer(String name, String topic, String body) {
        return () -> {
            log.info("线程： {} 监听队列：{}， topic: {}, body: {}, 开始处理", Thread.currentThread().getName()
                    , name, topic, body);
            RLock lock = redissonClient.getLock(LOCK_PREFIX + topic);
            try {
                lock.lock();
                IQueueListener queueListener = listenerMap.get(topic);
                if (ObjectUtil.isNull(queueListener)) {
                    log.error("topic {} 没有找到对应的监听器", topic);
                    return;
                }
                queueListener.consumer(body);
            } catch (Exception e) {
                log.error("消费失败");
            } finally {
                lock.unlock();
            }
        };
    }

    protected abstract Pair<String, String> take() throws InterruptedException;
}
```

在抽象类的静态代码块中，获取所有实现了IQueueListener接口的类。获取这些类中QueueConsumer注解里标明的topic，把topic和QueueListener绑定——存入lsitenerMap容器中。

在唯一的带参构造器中，先给当前的实现类的name属性赋值，然后再开启一个线程去监听延时队列，调用子类实现的take()获取队列中的值，如果取得到值就开启一个线程去调用consumer()消费。

在consumer()中，首先通过消息体Pair类的key值（topic）从listenerMap中获取对应的IQueueListener类，再由该类去处理消息体。

***

take()由具体的子类来实现

```java
@Slf4j
public class DelayQueue extends AbstractQueue {

    private RDelayedQueue<Pair<String, String>> delayedQueue;
    private RBlockingDeque<Pair<String, String>> blockingDeque;

    public DelayQueue(String name, RDelayedQueue<Pair<String, String>> delayedQueue,
                      RBlockingDeque<Pair<String, String>> blockingDeque) {
        super(name);
        this.blockingDeque = blockingDeque;
        this.delayedQueue = delayedQueue;
    }

    public static DelayQueue create(String name) {
        RBlockingDeque<Pair<String, String>> blockingDeque = redissonClient.getBlockingDeque(name);
        RDelayedQueue<Pair<String, String>> delayedQueue = redissonClient.getDelayedQueue(blockingDeque);
        return new DelayQueue(name, delayedQueue, blockingDeque);
    }

    public <T> void offer(String topic, T body, long delay) {
        delayedQueue.offer(Pair.of(topic, JSONUtil.toJsonStr(body)), delay, TimeUnit.SECONDS);
    }

    @Override
    protected Pair<String, String> take() throws InterruptedException {
        return blockingDeque.take();
    }
}
```

子类中提供了一个create()的静态方法，暴露给外部来创建延时队列。其中通过redissonClient创建了RBlockingDeque和RDelayedQueue，然后再调用构造方法。在子类的构造方法中，调用抽象类的构造方法，然后把给RBlockingDeque和RDelayedQueue两个成员变量赋值。

offer()就是新增延时任务，往RDelayedQueue中添加值。

take()就是获取延时任务，从RBlockingDeque获取值，在父类中的构造函数中开启的监听线程中被调用。

***

以上就通过redisson实现了延时队列，下面是使用用例。

```java
public class DelayClient {

    private static DelayQueue delayQueue = DelayQueue.create("DELAY_QUEUE_01");

    public static <T> void offer(String topic, T body, long delay) {
        delayQueue.offer(topic, body, delay);
    }

    public static <T> void offer(String topic, T body, Date date) {
        delayQueue.offer(topic, body, DateUtil.between(new Date(), date, DateUnit.SECOND));
    }
}
```

***

> tip: [项目源码](https://github.com/Alexhuihui/common-redis)