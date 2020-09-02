# 1.为什么要使用分布式锁

使用分布式锁的目的，无外乎就是保证同一时间只有一个客户端可以对共享资源进行操作。

## 1.1举一个很长的例子

系统 A 是一个电商系统，目前是一台机器部署，系统中有一个用户下订单的接口，但是用户下订单之前一定要去检查一下库存，确保库存足够了才会给用户下单。由于系统有一定的并发，所以会预先将商品的库存保存在 Redis 中，用户下单的时候会更新 Redis 的库存。此时系统架构如下：

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image002.jpg)

但是这样一来会产生一个问题：假如某个时刻，Redis 里面的某个商品库存为 1。

此时两个请求同时到来，其中一个请求执行到上图的第 3 步，更新数据库的库存为 0，但是第 4 步还没有执行。

而另外一个请求执行到了第 2 步，发现库存还是 1，就继续执行第 3 步。这样的结果，是导致卖出了 2 个商品，然而其实库存只有 1 个。

很明显不对啊！这就是典型的**库存超卖问题**。此时，我们很容易想到解决方案：用锁把 2、3、4 步锁住，让他们执行完之后，另一个线程才能进来执行第 2 步。

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image003.jpg)

按照上面的图，在执行第 2 步时，使用 Java 提供的 Synchronized 或者 ReentrantLock 来锁住，然后在第 4 步执行完之后才释放锁。

这样一来，2、3、4 这 3 个步骤就被“锁”住了，多个线程之间只能**串行化执行**。

 

当整个系统的并发飙升，一台机器扛不住了。现在要增加一台机器，如下图：

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image005.jpg)

增加机器之后，系统变成上图所示，假设此时两个用户的请求同时到来，但是落在了不同的机器上，那么这两个请求是可以同时执行了，还是会出现库存超卖的问题。

因为上图中的两个 A 系统，运行在两个不同的 JVM 里面，他们加的锁只对属于自己 JVM 里面的线程有效，对于其他 JVM 的线程是无效的。

因此，这里的问题是：Java 提供的原生锁机制在多机部署场景下失效了，这是因为两台机器加的锁不是同一个锁（两个锁在不同的 JVM 里面）。

那么，我们只要保证两台机器加的锁是同一个锁，问题不就解决了吗？此时，就该分布式锁隆重登场了。

分布式锁的思路是：**在整个系统提供一个全局、唯一的获取锁的“东西”，然后每个系统在需要加锁时，都去问这个“东西”拿到一把锁，这样不同的系统拿到的就可以认为是同一把锁。**

至于这个“东西”，可以是 Redis、Zookeeper，也可以是数据库。此时的架构如图：

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image007.jpg)

通过上面的分析，我们知道了库存超卖场景在分布式部署系统的情况下使用 Java 原生的锁机制无法保证线程安全，所以我们需要用到分布式锁的方案。

# 2.高效的分布式锁

在设计分布式锁的时候，应该考虑分布式锁至少要满足的一些条件，同时考虑如何高效的设计分布式锁，以下几点是必须要考虑的：

**(1)**  **互斥**

在分布式高并发的条件下，最需要保证在同一时刻只能有一个线程获得锁，这是最基本的一点。

**(2)**  **防止死锁**

在分布式高并发的条件下，比如有个线程获得锁的同时，还没有来得及去释放锁，就因为系统故障或者其它原因使它无法执行释放锁的命令,导致其它线程都无法获得锁，造成**死锁**。所以分布式非常有必要设置锁的有效时间，确保系统出现故障后，在一定时间内能够主动去释放锁，避免造成死锁的情况。

**(3)**  **性能**

对于访问量大的共享资源，需要考虑减少锁等待的时间，避免导致大量线程阻塞。

所以在锁的设计时，需要考虑两点。

1、 锁的颗粒度要尽量小。比如你要通过锁来减库存，那这个锁的名称你可以设置成是商品的ID,而不是任取名称。这样这个锁只对当前商品有效,锁的颗粒度小。

2、 锁的范围尽量要小。比如只要锁2行代码就可以解决问题的，那就不要去锁10行代码了。

**(4)**  **重入**

我们知道ReentrantLock是可重入锁，那它的特点就是：同一个线程可以重复拿到同一个资源的锁。重入锁非常有利于资源的高效利用。关于这点之后会做演示。

 

# 3.基于Redis实现分布式锁

## 3.1 使用Redis命令实现分布式锁

### 3.1.1加锁

加锁实际上就是在redis中，给Key键设置一个值，为避免死锁，并给定一个过期时间。

使用的命令**：SET lock_key random_value NX PX 5000**

值得注意的是：

random_value 是客户端生成的唯一的字符串。

NX 代表只在键不存在时，才对键进行设置操作。

PX 5000 设置键的过期时间为5000毫秒。

 

也可以使用另外一条命令:**SETNX key value** 

只不过过期时间无法设置。

 

这样，如果上面的命令执行成功，则证明客户端获取到了锁。

### 3.1.2解锁

解锁的过程就是将Key键删除，但要保证安全性，举个例子：客户端1的请求不能将客户端2的锁给删除掉。

释放锁涉及到两条指令，这两条指令不是原子性的,需要用到redis的lua脚本支持特性，redis执行lua脚本是原子性的。脚本如下：

```lua
if redis.call('get',KEYS[1]) == ARGV[1] then 

  return redis.call('del',KEYS[1]) 

else

  return 0 

end
```

 

这种方式比较简单，但是也有一个最重要的问题：**锁不具有可重入性**。

 

## 3.2使用Redisson实现分布式锁

### 3.2.1Redisson介绍

[Redisson](https://redisson.org/)是架设在[Redis](http://redis.cn/)基础上的一个Java驻内存数据网格（In-Memory Data Grid）。充分的利用了Redis键值数据库提供的一系列优势，基于Java实用工具包中常用接口，为使用者提供了一系列具有分布式特性的常用工具类。使得原本作为协调单机多线程并发程序的工具包获得了协调分布式多机多线程并发系统的能力，大大降低了设计和研发大规模分布式系统的难度。同时结合各富特色的分布式服务，更进一步简化了分布式环境中程序相互之间的协作。

 

### 3.2.2Redisson简单使用

```java
Config config = new Config(); 

config.useClusterServers() 

.addNodeAddress("redis://192.168.31.101:7001") 

.addNodeAddress("redis://192.168.31.101:7002") 

.addNodeAddress("redis://192.168.31.101:7003") 

.addNodeAddress("redis://192.168.31.102:7001") 

.addNodeAddress("redis://192.168.31.102:7002") 

.addNodeAddress("redis://192.168.31.102:7003"); 

 

RedissonClient redisson = Redisson.create(config); 

RLock lock = redisson.getLock("anyLock"); 

lock.lock(); 

lock.unlock(); 
```

只需要通过它的 API 中的 Lock 和 Unlock 即可完成分布式锁，而且考虑了很多细节：

l Redisson 所有指令都通过 Lua 脚本执行，Redis 支持 **Lua 脚本原子性执行**。

l Redisson 设置一个 Key 的默认过期时间为 30s，但是如果获取锁之后，会有一个WatchDog每隔10s将key的超时时间设置为30s。

 

另外，Redisson 还提供了对 Redlock 算法的支持，它的用法也很简单：

```java
RedissonClient redisson = Redisson.create(config); 

RLock lock1 = redisson.getFairLock("lock1"); 

RLock lock2 = redisson.getFairLock("lock2"); 

RLock lock3 = redisson.getFairLock("lock3"); 

RedissonRedLock multiLock = new RedissonRedLock(lock1, lock2, lock3); 

multiLock.lock(); 

multiLock.unlock(); 
```

 

### 3.2.3Redisson原理分析

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image009.jpg)

#### (1)  加锁机制

线程去获取锁，获取成功: 执行lua脚本，保存数据到redis数据库。

线程去获取锁，获取失败: 一直通过while循环尝试获取锁，获取成功后，执行lua脚本，保存数据到redis数据库。

#### (2)  WatchDog自动延期机制

在一个分布式环境下，假如一个线程获得锁后，突然服务器宕机了，那么这个时候在一定时间后这个锁会自动释放，也可以设置锁的有效时间(不设置默认30秒），这样的目的主要是防止死锁的发生。但是在实际情况中会有一种情况，业务处理的时间可能会大于锁过期的时间，这样就可能**导致解锁和加锁不是同一个线程。**所以WatchDog作用就是Redisson实例关闭前，不断延长锁的有效期。

如果程序调用加锁方法显式地给了有效期，是不会开启后台线程(也就是watch dog)进行延期的，如果没有给有效期或者给的是-1，redisson会默认设置30s有效期并且会开启后台线程(watch dog)进行延期

多久进行一次延期：**（默认有效期/3）**，默认有效期可以设置修改的，即默认情况下每隔10s设置有效期为30s

#### (3)  可重入加锁机制

Redisson可以实现可重入加锁机制的原因：

l Redis存储锁的数据类型是Hash类型

l Hash数据类型的key值包含了当前线程的信息

下面是redis存储的数据
 ![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image011.jpg)

这里表面数据类型是Hash类型,Hash类型相当于我们java的 <key,<key1,value>> 类型,这里key是指 'redisson'

它的有效期还有9秒，我们再来看里们的key1值为078e44a3-5f95-4e24-b6aa-80684655a15a:45它的组成是:

guid + 当前线程的ID。后面的value是就和可重入加锁有关。value代表同一客户端调用lock方法的次数，即可重入计数统计。

 

**举图说明**

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image013.jpg)

上面这图的意思就是可重入锁的机制，它最大的优点就是相同线程不需要在等待锁，而是可以直接进行相应操作。

### 3.2.4 获取锁的流程

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image015.jpg)

其中的指定字段也就是hash结构中的field值（构成是uuid+线程id），即判断锁是否是当前线程

### 3.2.5 加锁的流程

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image016.png)

### 3.2.6 释放锁的流程

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image018.jpg)

# 4. 使用Redis做分布式锁的缺点

Redis有三种部署方式

l 单机模式

l Master-Slave+Sentienl选举模式

l Redis Cluster模式

 

如果采用单机部署模式，会存在单点问题，只要 Redis 故障了。加锁就不行了

 

采用 Master-Slave 模式，加锁的时候只对一个节点加锁，即便通过 Sentinel 做了高可用，但是如果 Master 节点故障了，发生主从切换，此时就会有可能出现锁丢失的问题。

 

基于以上的考虑，Redis 的作者也考虑到这个问题，他提出了一个 RedLock 的算法。

这个算法的意思大概是这样的：假设 Redis 的部署模式是 Redis Cluster，总共有 5 个 Master 节点。

通过以下步骤获取一把锁：

- 获取当前时间戳，单位是毫秒。
- 轮流尝试在每个 Master 节点上创建锁，过期时间设置较短，一般就几十毫秒。
- 尝试在大多数节点上建立一个锁，比如 5 个节点就要求是 3 个节点（n / 2 +1）。
- 客户端计算建立好锁的时间，如果建立锁的时间小于超时时间，就算建立成功了。
- 要是锁建立失败了，那么就依次删除这个锁。
- 只要别人建立了一把分布式锁，你就得不断轮询去尝试获取锁。

但是这样的这种算法，可能会出现**节点崩溃重启，多个客户端持有锁**等其他问题，无法保证加锁的过程一定正确。例如：

假设一共有5个Redis节点：A, B, C, D, E。设想发生了如下的事件序列：

(1)客户端1成功锁住了A, B, C，获取锁成功（但D和E没有锁住）。

(2)节点C崩溃重启了，但客户端1在C上加的锁没有持久化下来，丢失了。

(3)节点C重启后，客户端2锁住了C, D, E，获取锁成功。

这样，客户端1和客户端2同时获得了锁（针对同一资源）。

 

 

# 5.在Springboot中集成redisson

## 5.1 引入依赖

```xml
<dependency>

    <groupId>org.redisson</groupId>

    <artifactId>redisson-spring-boot-starter</artifactId>

    <version>3.13.3</version>

</dependency>
```

 

如果引入以上的依赖，会丧失一定的配置灵活性

 

## 5.2 application.yml

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image020.jpg)

简单配置一下，此时启动应用就已经将redisson集成到了springboot中，此时的配置就是集群环境下的。如果需要单机和集群的切换，则需要引入redisson的依赖，自己编写配置类，这里为了简单，采用starter的方式集成

 

## 5.3测试

```java
import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.TimeUnit;

/**
 * @Author Song
 * @Date 2020/8/31 14:49
 * @Version 1.0
 * @Description
 */
@Slf4j
@RestController
public class TestController {
    private static final String KEY = "mylock";

    @Autowired
    private RedissonClient redissonClient;


    @GetMapping("/lock1")
    public String testLock1() {
        log.info("lock1  正在获取锁。。。。");
        RLock lock = redissonClient.getLock(KEY);
        lock.lock();
        log.info(Thread.currentThread().getName() + ":" + Thread.currentThread().getId() + " lock1 已经获取到锁");
        try {
            //模拟业务处理20s
            log.info("正在进行业务处理");
            TimeUnit.SECONDS.sleep(20);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        lock.unlock();
        log.info(Thread.currentThread().getName() + ":" + Thread.currentThread().getId() + " lock1 已解锁");
        return "lock1";
    }

    @GetMapping("/lock2")
    public String testLoc2() {
        log.info("lock2  正在获取锁。。。。");
        RLock lock = redissonClient.getLock(KEY);
        lock.lock();
        log.info(Thread.currentThread().getName() + ":" + Thread.currentThread().getId() + " lock2 已经获取到锁");
        try {
            //模拟业务处理20s
            log.info("正在进行业务处理");
            TimeUnit.SECONDS.sleep(20);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        lock.unlock();
        log.info(Thread.currentThread().getName() + ":" + Thread.currentThread().getId() + " lock2 已解锁");
        return "lock2";
    }
}

```

 

然后平行地启动两个实例，分别访问localhost:8080/lock1和localhost:8081/lock2

模拟分布式访问。

启动效果如图：

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image022.jpg)

当同时访问时，8080的实例会进行业务处理，8081的实例会阻塞，如图：

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image024.jpg)

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image026.jpg)

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image028.jpg)

此时Redis中存储的hash的key值与8080实例中线程id一致

 

当8080实例中释放锁之后，8081实例中会立刻获取到锁并进行下一步的模拟业务，如图：

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image030.jpg)

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image032.jpg)

![img](D:\MyStudy\学习杂记\Redis\使用Redis实现分布式锁.assets\clip_image034.jpg)

此时Redis中存储的hash的key值和8081实例中的线程id一致。

 

到最后8081实例中的锁释放之后，在Redis中找不到key值。实现了分布式锁