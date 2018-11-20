---
layout: post
title: "基于redis的分布式延时队列"
description: "基于redis的分布式延时队列"
categories: [queue]
tags: [queue]
redirect_from:
  - /2018/05/30/
---
延时队列的使用场景非常多，目前集中式场景可以使用JDK自带的delayQueue，本文使用redis队列来实现两种分布式的延时队列，并进行了对比分析

# 1 封装统一的队列消息结构
```
public class DelayMessage {

    /**
     * 正执行消息的token
     */
    private String tmpKey;

    /**
     * 消息内容
     */
    private String message;

    /**
     * 延迟时间 纳秒
     */
    private long delay;

    /**
     * 到期时间 纳秒
     */
    private long expire;

    /**
     * 创建时间 纳秒
     */
    private long registerTime;
    
    ...
}
```

# 2 同步延时队列
原理：通过LRANG获取队列头部元素，从而获得到期时间，当数据达到到期时间后，通过LPOP取出数据。并发情况下会加分布式锁，同一时刻只能有一个线程访问队列并阻塞等待数据准备完成(达到延时时间)。当线程获取到可以返回的数据后，才能释放锁。
```
public DelayMessage pop() {
    while (true) {
        Long waitTime = null;
        Jedis jedis = jedisPool.getResource();
        try {
            if (this.syn) {
                lock.lock(this.queueName);
            }
            //获取头部的数据
            List<String> dataList = jedis.lrange(this.queueName, 0L, 0L);
            if (CollectionUtils.isNotEmpty(dataList)) {
                String data = dataList.get(0);
                DelayMessage message = gson.fromJson(data, DelayMessage.class);
                if (message != null) {
                    long now = System.nanoTime();
                    if (message.getExpire() > now) {
                        //没有到期
                        waitTime = message.getExpire() - now;
                    } else {
                        jedis.lpop(this.queueName);
                        return message;
                    }
                }
            }

            if (waitTime != null) {
                try {
                    Thread.sleep(waitTime / 1000);
                } catch (InterruptedException e) {
                    //do nothing
                }
            }

        } finally {
            if (this.syn) {
                lock.unlock(this.queueName);
            }
            jedis.close();
        }
    }
}
```
- 基于redis，**并发情况下会加分布式锁**，单线程场景（syn=false）性能较好， 并发场景性能较差
- **若在并发场景下，设置syn=false，会导致消息重复消费、消息丢失的情况**
- 支持delay时间的动态调整

# 3 并发延时队列
原理：直接通过LPOP取出数据，当数据达到到期时间后立即返回，若未达到到期时间，线程sleep至到期时间后将数据返回。这样就允许队列在无分布式锁的情况下并行消费。
存在的问题：从数据LPOP移除队列到数据返回之间有一段时间间隔，如果系统在这段时间出现异常，比如宕机、重启等等，就会出现数据丢失的情况。为了避免此种情况，引入了ack的方式，在通过LPOP去除数据后，随机将数据存储到另外一个临时redis set中，在用户接受到消息并处理完成后手动执行ack方法后才将数据从临时set中移除。而在机器重启时，再将改临时set里面的数据重新put到延时队列中。
```
public DelayMessage pop() {
    while (true) {
        Long waitTime;
        Jedis jedis = jedisPool.getResource();
        try {
            // 取队列头部的消息
            String result = jedis.lpop(queueName);
            // 队列非空
            if (result != null) {
                //暂存此条消息，放入执行集合
                String tmpKey = null;
                if (!autoAck) {
                    tmpKey = this.setExeMsg(result);//解决宕机数据丢失
                }
                DelayMessage delayMessage = gson.fromJson(result, DelayMessage.class);
                if (delayMessage != null) {
                    if (delayMessage.getExpire() > System.nanoTime()) {
                        // 消息未到可执行状态,休眠等待
                        waitTime = delayMessage.getExpire() - System.nanoTime();
                        try {
                            Thread.sleep(waitTime);
                        } catch (InterruptedException e) {
                            // do nothing
                        }
                    }
                    delayMessage.setTmpKey(tmpKey);
                    return delayMessage;
                }
            }
        } finally {
            jedis.close();
        }
    }
}
```

- 基于redis，支持在**无分布式锁**的情况下进行并发消费
- autoAck为true时，吞吐量性能极好，autoAck为false，吞吐量会稍有下降
- 支持delay时间的动态调整
- **autoAck为false时，必须在处理完消息后手动调用ack方法，否则会导致应用重启后重新开始消费**

# 4 性能对比

* RedisConcurrentDelayQueue和RedisSynDelayQueue的简单对比，数据是线下单机环境测试数据

| 队列种类                     |  消费线程数| syn | autoAck |  耗时    |  消息丢失  | 重复消费   |
|:---------------------------:|:----------:|:----------:|:-------:|:--------:|:----------:|:----------:|
| RedisConcurrentDelayQueue         |  1         | -          |  false  | 53936ms  |   无       |     无     |
| RedisConcurrentDelayQueue         |  1         | -          |  true   | 13130ms  | 消费进程关闭，正在处理的消息会丢失 |   无   |
| RedisSynDelayQueue             |  1         | true       |   -     | 55420ms  |   无       |   无   |
| RedisSynDelayQueue             |  1         | false      |   -     | 20012ms  |   无       |   无   |
| RedisConcurrentDelayQueue         |  10        | -          |  false  | 7279ms  |   无       |     无     |
| RedisConcurrentDelayQueue         |  10        | -          |  true   | 1181ms  | 消费进程关闭，正在处理的消息会丢失 |   无   |
| RedisSynDelayQueue             |  10        | true       |   -     | 61532ms  |   无       |   无   |
| RedisSynDelayQueue             |  10        | false      |   -     | -  |   大量消息丢失       |   大量重复消费   |

1. 若能接受系统重启、关闭时的少量消息丢失，推荐RedisConcurrentDelayQueue，并设置autoAck为true：性能最好，且消费线程越多，消费速度（吞吐量）也会相对越好
2. 若不能接受消息丢失，在单机、单线程消费的场景下，可以选择RedisConcurrentDelayQueue（autoAck设置为false）RedisSynDelayQueue（syn设置为false）；
3. 若不能接受消息丢失，且需要在多线程、分布式场景下消费，推荐推荐RedisConcurrentDelayQueue（autoAck设置为false），消费线程越多，消费速度（吞吐量）也会相对越好；
4. RedisSynDelayQueue在并发消费的场景下性能较差，不推荐使用。

# 5 项目代码
代码托管在Github上，并附有使用demo，欢迎下载运行: https://github.com/Cuner/delay-queue

# 6 切换思路
当有延时任务或者延时消息产生时，暂时将延时任务/消息存放至某个容器里面(可以是数据记录的方式存储在db，也可以存储在redis zset中等等)，然后采用定时器不断去轮训容器里面的各个延时任务/消息，当扫描获取到达到过期时间的延时任务/消息后，将其推送到消息队列中以供消费，推送成功后再将其从容器里移除。
值得注意的是，同一个达到到期时间的延时任务/消息，可能会被连续的两次扫描命中（延时任务/消息从容器移除有延时），最后导致消息重复消费。

参考：[有赞延迟队列设计](https://tech.youzan.com/queuing_delay/)