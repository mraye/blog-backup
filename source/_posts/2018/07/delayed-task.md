---
title: 延时任务
date: 2018-07-15 13:57:13
tags: [延时任务,并发]
---



项目需求中经常会碰到:

+ 定单30分钟未付款自动取消
+ 交易成功后发送通知，如果首次没有发送成功，则延迟1分钟后再次发送，如果不没有通知成功，则按照延迟1,3,5,8,..分钟后继续通知



目前的解决方案有：
+ 数据库轮询
+ Jdk自带的`DelayQueue`
+ 时间轮(循环队列)，
+ redis中的有序集合`(Zset)`,
+ redis中的`Keyspace Notifications`

<!-- more -->

### 数据库轮询

**思路:**
> 通过定时任务扫描数据库中过期的数据，取出数据，将定单的状态调整为无效


**解决方案:**
通常使用`quartz`来实现:
+ 分布式定时任务
+ 任务持久化，服务宕机重启之后任务仍然有效

缺点:
+ 数据库和服务器压力大
+ 实时性不高


### Jdk自带的DelayQueue

`DelayQueue`是一个无界的阻塞队列，放入`DelayQueue`中的对象必须实现`Delayed`接口，该接口只有一个方法:  

```java
public interface Delayed extends Comparable<Delayed> {

    //在给定的时间，返回这个对象剩余的时间
    long getDelay(TimeUnit unit);
}

```

优点和缺点:  

+ 单机版，基于内存，一旦宕机，数据不将存在，需要自行实现持久化
+ 如果数据量大，内存消耗大;如果失效时间太长，常驻内存时间也久，服务器运行不流畅，系统卡顿
+ 定单在某时间点量大，很容易造成OOM

### redis中zset(有序集合)

**思路:**  
> 生成定单的时候通过reddis的`zset`方法加入到有序的集合中,然后启动一个线程，不停的扫描zset集合，通过`zrange`或者`zrangeWithScores`命令取出`score`最低的定单编号，然后再通过`zrem`命令将失效定单从zset移除，这里的`score`就是**定单失效的时间戳**  


```java
public class OrderTest extends BaseTest{

    @Autowired
    private JedisPool jedisPool;

    @Test
    public void delayTask() throws InterruptedException {
        final  Jedis jedis = jedisPool.getResource();
        final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
       for (int i=0; i < 5; i++){
           Calendar calendar = Calendar.getInstance();
           calendar.add(Calendar.SECOND, new Random().nextInt(10)+3);
           long score = calendar.getTimeInMillis();
           jedis.zadd("order", score, "orderId-0000" + i);
           System.out.println("生成定单: orderId-0000"+ i +" 过期时间: "+sdf.format(calendar.getTime()));
       }


        Thread thread = new Thread(()->{
           while (true){
               Set<Tuple> invalidList = jedis.zrangeWithScores("order",0,1);
               if (invalidList == null || invalidList.isEmpty()){
                   System.out.println("暂时无失效定单...");
                   try {
                       TimeUnit.SECONDS.sleep(2);
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
                   continue;
               }

               long score = (long)((Tuple)invalidList.toArray()[0]).getScore();
               Calendar calendar = Calendar.getInstance();
               long now = calendar.getTime().getTime();
               if (now >= score){
                    String orderId = ((Tuple)invalidList.toArray()[0]).getElement();
                    jedis.zrem("order", orderId);
                   System.out.println("处理失效定单编号: "+ orderId+ " 失效时间是: "+ sdf.format(new Date(score))+" 当前时间:" + sdf.format(calendar.getTime()));
               }

           }
        });
        thread.start();
        thread.join();

    }

}

```


结果:  

```console
生成定单: orderId-00000 过期时间: 2018-07-15 16:06:03
生成定单: orderId-00001 过期时间: 2018-07-15 16:06:04
生成定单: orderId-00002 过期时间: 2018-07-15 16:06:06
生成定单: orderId-00003 过期时间: 2018-07-15 16:06:11
生成定单: orderId-00004 过期时间: 2018-07-15 16:06:09
处理失效定单编号: orderId-00000 失效时间是: 2018-07-15 16:06:03 当前时间:2018-07-15 16:06:05
处理失效定单编号: orderId-00001 失效时间是: 2018-07-15 16:06:04 当前时间:2018-07-15 16:06:06
处理失效定单编号: orderId-00002 失效时间是: 2018-07-15 16:06:06 当前时间:2018-07-15 16:06:07
处理失效定单编号: orderId-00004 失效时间是: 2018-07-15 16:06:09 当前时间:2018-07-15 16:06:09
处理失效定单编号: orderId-00003 失效时间是: 2018-07-15 16:06:11 当前时间:2018-07-15 16:06:11
暂时无失效定单...
暂时无失效定单...
暂时无失效定单...
暂时无失效定单...
```



但是这样有一个问题，**就是在多线程环境下，多个线程可以取出相同的重复的失效定单编号**


`OrderService.java`负责定单逻辑:  

```java
package com.sxz.zxd.order;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.Tuple;

import java.text.SimpleDateFormat;
import java.util.Calendar;
import java.util.Date;
import java.util.Random;
import java.util.Set;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

/**
 * Created  on 2018/7/15.
 */

public class OrderService {

    private JedisPool jedisPool;

    public OrderService(JedisPool jedisPool) {
        this.jedisPool = jedisPool;
    }

    public void produceOrder(){
        Jedis jedis = jedisPool.getResource();
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

        for (int i=0; i < 5; i++){
            Calendar calendar = Calendar.getInstance();
            long score = calendar.getTimeInMillis();
            jedis.zadd("order", score, "orderId-0000"+i);
            System.out.println("生成定单: orderId-0000"+i+" 过期时间: "+sdf.format(calendar.getTime()));
        }
    }


    public void consumeInvalidOrder() throws InterruptedException {

        Jedis jedis = jedisPool.getResource();
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        Set<Tuple> invalidList = jedis.zrangeWithScores("order", 0, 1);
        if (invalidList == null || invalidList.isEmpty()){
            System.out.println("暂时无失效定单...");
            return;
        }
        long score = (long)((Tuple)invalidList.toArray()[0]).getScore();
        Calendar calendar = Calendar.getInstance();
        long now = calendar.getTime().getTime();
        if (now >= score){
            String orderId = ((Tuple)invalidList.toArray()[0]).getElement();
            jedis.zrem("order", orderId);
            System.out.println("处理失效定单编号: "+ orderId+ " 失效时间是: "+ sdf.format(new Date(score))+" 当前时间:" + sdf.format(calendar.getTime()));
        }
    }

}

```

测试:  

```java
@Test
public void delayTask2() throws InterruptedException {
    CountDownLatch countDownLatch = new CountDownLatch(1);
    OrderService orderService = new OrderService(jedisPool);
    orderService.produceOrder();
    Thread.sleep(2000);
    for (int i=0; i < 15; i++){
        executor.execute(()->{
            try {
                countDownLatch.await();
                orderService.consumeInvalidOrder();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
    countDownLatch.countDown();
    Thread.sleep(10000);

}

```

结果:  

```console
生成定单: orderId-00000 过期时间: 2018-07-15 17:13:55
生成定单: orderId-00001 过期时间: 2018-07-15 17:13:55
生成定单: orderId-00002 过期时间: 2018-07-15 17:13:55
生成定单: orderId-00003 过期时间: 2018-07-15 17:13:55
生成定单: orderId-00004 过期时间: 2018-07-15 17:13:55
处理失效定单编号: orderId-00000 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:13:58
处理失效定单编号: orderId-00000 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:13:58
处理失效定单编号: orderId-00000 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:13:58
处理失效定单编号: orderId-00000 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:13:58
处理失效定单编号: orderId-00000 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:13:58
处理失效定单编号: orderId-00001 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:13:59
处理失效定单编号: orderId-00001 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:13:59
处理失效定单编号: orderId-00002 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:13:59
处理失效定单编号: orderId-00002 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:13:59
处理失效定单编号: orderId-00002 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:13:59
处理失效定单编号: orderId-00003 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:14:00
处理失效定单编号: orderId-00003 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:14:00
处理失效定单编号: orderId-00004 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:14:00
处理失效定单编号: orderId-00004 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:14:00
处理失效定单编号: orderId-00004 失效时间是: 2018-07-15 17:13:55 当前时间:2018-07-15 17:14:00
....
```

可以发现，15个线程并发，同时处理了相同的失效定单编号。。。这样肯定有问题的哇，，，



**一种方法是使用redis中的分布式锁`setnx`命令，一种方法是判断`jedis.zrem`操作返回的数据，如果大于0，则说明`zset`中原来失效的定单已经删除成功，反之，则说明失效定单已经被处理了**



只需要修改`OrderService`中的`consumeInvalidOrder`方法:  

```java
public void consumeInvalidOrder() throws InterruptedException {

     Jedis jedis = jedisPool.getResource();
     SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
     Set<Tuple> invalidList = jedis.zrangeWithScores("order", 0, 1);
     if (invalidList == null || invalidList.isEmpty()){
         System.out.println("暂时无失效定单...");
         return;
     }
     long score = (long)((Tuple)invalidList.toArray()[0]).getScore();
     Calendar calendar = Calendar.getInstance();
     long now = calendar.getTime().getTime();
     if (now >= score){
         String orderId = ((Tuple)invalidList.toArray()[0]).getElement();
         Long result = jedis.zrem("order", orderId);
//            System.out.println("处理失效定单编号: "+ orderId+ " 失效时间是: "+ sdf.format(new Date(score))+" 当前时间:" + sdf.format(calendar.getTime()));
         if (result!=null && result.intValue() > 0) {
             System.out.println("处理失效定单编号: "+ orderId+ " 失效时间是: "+ sdf.format(new Date(score))+" 当前时间:" + sdf.format(calendar.getTime()));
         }
     }
 }

```

测试案例不变,结果:  

```bash
生成定单: orderId-00000 过期时间: 2018-07-15 17:24:57
生成定单: orderId-00001 过期时间: 2018-07-15 17:24:57
生成定单: orderId-00002 过期时间: 2018-07-15 17:24:57
生成定单: orderId-00003 过期时间: 2018-07-15 17:24:57
生成定单: orderId-00004 过期时间: 2018-07-15 17:24:57
处理失效定单编号: orderId-00000 失效时间是: 2018-07-15 17:24:57 当前时间:2018-07-15 17:24:59
处理失效定单编号: orderId-00001 失效时间是: 2018-07-15 17:24:57 当前时间:2018-07-15 17:24:59
处理失效定单编号: orderId-00002 失效时间是: 2018-07-15 17:24:57 当前时间:2018-07-15 17:24:59
处理失效定单编号: orderId-00003 失效时间是: 2018-07-15 17:24:57 当前时间:2018-07-15 17:24:59
暂时无失效定单...
处理失效定单编号: orderId-00004 失效时间是: 2018-07-15 17:24:57 当前时间:2018-07-15 17:24:59
暂时无失效定单...
```

### 时间轮

待续...

### redis中的`Keyspace Notifications`

待续...
