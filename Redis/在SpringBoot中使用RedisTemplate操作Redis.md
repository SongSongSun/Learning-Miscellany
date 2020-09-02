**在SpringBoot中使用RedisTemplate操作Redis**

# 1.Spring-Redis介绍

spring-data-redis 针对jedis提供了如下功能：

(1)  连接池自动管理，提供了一个高度封装的“RedisTemplate”类

(2)  针对jedis客户端中大量api进行了归类封装,将同一类型操作封装为operation接口

a)   ValueOperations：简单K-V操作

b)   SetOperations：set类型数据操作

c)   ZSetOperations：zset类型数据操作

d)   HashOperations：针对map类型的数据操作

e)   ListOperations：针对list类型的数据操作

(3)  提供了对key的“bound”(绑定)便捷化操作API，可以通过bound封装指定的key，然后进行一系列的操作而无须“显式”的再次指定Key，即BoundKeyOperations：

a)   BoundValueOperations

b)   BoundSetOperations

c)   BoundListOperations

d)   BoundSetOperations

e)   BoundHashOperations

(4)  将事务操作封装，有容器控制。

(5)  针对数据的“序列化/反序列化”，提供了多种可选择策略(RedisSerializer)

a)   JdkSerializationRedisSerializer：POJO对象的存取场景，使用JDK本身序列化机制，将pojo类通过ObjectInputStream/ObjectOutputStream进行序列化操作，最终redis-server中将存储字节序列。是目前最常用的序列化策略。

b)   StringRedisSerializer：Key或者value为字符串的场景，根据指定的charset对数据的字节序列编码成string，是“new String(bytes, charset)”和“string.getBytes(charset)”的直接封装。是最轻量级和高效的策略。

c)   JacksonJsonRedisSerializer：jackson-json工具提供了javabean与json之间的转换能力，可以将pojo实例序列化成json格式存储在redis中，也可以将json格式的数据转换成pojo实例。因为jackson工具在序列化和反序列化时，需要明确指定Class类型，因此此策略封装起来稍微复杂。【需要jackson-mapper-asl工具支持】

d)   OxmSerializer：提供了将javabean与xml之间的转换能力，目前可用的三方支持包括jaxb，apache-xmlbeans；redis存储的数据将是xml工具。不过使用此策略，编程将会有些难度，而且效率最低；不建议使用。【需要spring-oxm模块的支持】

 

# 2.关系型数据库的redis的key的设计

(1)  把表名转换为key前缀 如, tag:

(2)  第2段放置用于区分区key的字段--对应mysql中的主键的列名,如userid

(3)  第3段放置主键值,如2,3,4...., a , b ,c

(4)  第4段,写要存储的列名

例：user:userid:9:username

并且只要有可能就要利用key超时的优势

 

# 3.在SpringBoot中使用RedisTemplate

## 3.1 引入依赖

在项目中添加依赖

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-data-redis</artifactId>
 </dependency>
```

 

## 3.2 配置文件application.yml

### 3.2.1单机版配置文件

![img](file:///C:/Users/dell/AppData/Local/Temp/msohtmlclip1/01/clip_image002.jpg)

 

### 3.2.2集群版配置文件

配置集群需要添加依赖，否则将无法正确连接到redis

```xml
<dependency>
   <groupId>org.apache.commons</groupId>
   <artifactId>commons-pool2</artifactId>
 </dependency>
```

配置文件如下

![img](D:\MyStudy\学习杂记\Redis\在SpringBoot中使用RedisTemplate操作Redis.assets\clip_image004-1599010209087.jpg)

 

## 3.3 RedisTemplate配置

```java
package com.song.redistemplatedemo.config;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;

import java.net.UnknownHostException;

/**
 * @Author Song
 * @Date 2020/8/26 17:22
 * @Version 1.0
 * @Description
 */
@Configuration
public class RedisConfig {

    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        //使用Jackson2JsonRedisSerializer来序列化和反序列化redis的value值（默认使用JDK的序列化方式)
        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<Object>(Object.class);
        ObjectMapper om = new ObjectMapper();
        // 指定要序列化的域，field,get和set,以及修饰符范围，ANY是都有包括private和public
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        // 指定序列化输入的类型，类必须是非final修饰的，final修饰的类，比如String,Integer等会跑出异常
        om.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.WRAPPER_ARRAY);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
        // 配置连接工厂
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(jackson2JsonRedisSerializer);
        template.setValueSerializer(jackson2JsonRedisSerializer);
        // 设置hash key 和value序列化模式
        template.setHashKeySerializer(jackson2JsonRedisSerializer);
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }

    @Bean
    @ConditionalOnMissingBean(StringRedisTemplate.class)
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}


```

 

## 3.4测试文件

```java
package com.song.redistemplatedemo;

import org.junit.jupiter.api.Test;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.redis.core.*;

import java.util.ArrayList;
import java.util.List;

@SpringBootTest
class RedisTemplateDemoApplicationTests {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    @Autowired
    private RedissonClient redissonClient;

    @Test
    void listSet() {
        List<Object> demo = new ArrayList<>();
        demo.add("1");
        demo.add("2");
        Long springboot = redisTemplate.opsForList().rightPushAll("springboot", demo);
        System.out.println(springboot);
    }

    @Test
    void listGet() {
        List<Object> springboot = redisTemplate.opsForList().range("springboot", 0, -1);
        for (Object string : springboot) {
            System.out.println(string);
        }
    }

    @Test
    void stringSet() {
        redisTemplate.opsForValue().set("cluster1", "springboot");
        redisTemplate.opsForValue().set("cluster2", "java");
        redisTemplate.opsForValue().set("cluster3", "python");
    }

    @Test
    void stringGet() {
        System.out.println(redisTemplate.opsForValue().get("cluster1"));
        System.out.println(redisTemplate.opsForValue().get("cluster2"));
        System.out.println(redisTemplate.opsForValue().get("cluster3"));
    }
}

```

在需要使用Redis的地方，自动装配RedisTemplate即可。

 

## 3.5 Redis指令与RedisTemplate接口对应关系

| **数据类型** | **命令**                                                     | **对应redisTemplate**                                        | **命令说明**                                                 |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **String**   | set key value                                                | redisTemplate.opsForValue()  .set(K key, V value)            | 设定key的值                                                  |
|              | setex key seconds value                                      | redisTemplate.opsForValue()  .set(K key, V value, long timeout,  TimeUnit unit) | 设定key的值并设定过期时间                                    |
|              | Setnx key value                                              | redisTemplate.opsForValue()  .setIfAbsent(K key, V value)    | key 必须不存在，才能设置为value                              |
|              | Set key value xx                                             | redisTemplate.opsForValue()  .setIfPresent(K key, V value)   | Key必须存在才能设置成value，常用作更新                       |
|              | mset key value [key value ...]                               | redisTemplate.opsForValue()  .multiSet(Map<? extends K, ?  extends V> map) | 批量设置多对key-value                                        |
|              | get key                                                      | redisTemplate.opsForValue()  .get(Object key)                | 获取key对应的value                                           |
|              | mget key [key ...]                                           | redisTemplate.opsForValue()  .multiGet(Collection<K> keys)   | 批量获取对个key的值，值的顺序与key的顺序一致                 |
|              | incr key                                                     | redisTemplate.opsForValue()  .increment(K key)               | 自增key对应的值，如果值不是整数，返回错误，如果值不存在，从0开始自增 |
|              | decr key                                                     | redisTemplate.opsForValue()  .decrement(K key)               | 自减key对应的值，规则和自增一致                              |
|              | incrby key increment                                         | redisTemplate.opsForValue()  .increment(K key, double delta) | 将key对应的值增加指定的increment，规则与incr一致             |
|              | decrby key increment                                         | redisTemplate.opsForValue()  .decrement(K key, long delta)   | 将key对应的值减少指定的increment，规则与incr一致             |
| **Hash**     | hset key field value                                         | redisTemplate.opsForHash()  .put(H key, HK hashKey, HV value) | 为key设置field-value                                         |
|              | hmset key field value [field  value ...]                     | redisTemplate.opsForHash()  .putAll*(*H key, Map*<*?  extends HK, ? extends HV*>* m*)* | 批量为key设置多对field-value                                 |
|              | hget key field                                               | redisTemplate.opsForHash()  .get*(*H key, Object hashKey*)*  | 获取key的field对应的值                                       |
|              | hmget key field [field ...]                                  | redisTemplate.opsForHash()  .multiGet*(*Hkey,  Collection*<*HK*>* hashKeys*)* | 批量获取key下，多个field对应的值，顺序与field一致            |
|              | hdel key field [field ...]                                   | redisTemplate.opsForHash()  .delete*(*H key, Object... hashKeys*)* | 删除key下一个或多个field-value                               |
|              | hlen key                                                     | redisTemplate.opsForHash()  .size*(*H key*)*                 | 计算对应key的field-value的个数                               |
|              | hkeys key                                                    | redisTemplate.opsForHash()  .keys(H key)                     | 获取key对应的所有field集合                                   |
|              | hgetall key                                                  | redisTemplate.opsForHash()  .entries(H key)                  | 获取key下所有的field-value                                   |
|              | hexists key field                                            | redisTemplate.opsForHash()  .hasKey(H key, Object hashKey)   | 判断key下的某一个field是否存在                               |
| **List**     | rpush key value [value ...]                                  | redisTemplate.opsForList()  .rightPush(K key, V value)     redisTemplate.opsForList()  .rightPushAll(K key, Collection<V>  values) | 从key对应的列表尾部添加一个或者多个值                        |
|              | lpush key value [value ...]                                  | redisTemplate.opsForList()  .leftPush(K key, V value)     redisTemplate.opsForList()  .leftPushAll*(*K key, Collection*<*V*>*  values*)* | 从key对应的列表头部添加一个或者多个值                        |
|              | linsert key before\|after pivot  value                       | redisTemplate.opsForList()  .leftPush(K key, V pivot, V  value)     redisTemplate.opsForList()  .rightPush(K key, V pivot, V  value) | 从key对应的列表中的第一个pivot的前面或者后面添加一个value    |
|              | lindex key index                                             | redisTemplate.opsForList()  .index(K key, long index)        | 获取列表指定索引下的元素                                     |
|              | lrange key start stop                                        | redisTemplate.opsForList()  .range(K key, long start, long  end) | 获取列表指定范围内的元素                                     |
|              | lpop key                                                     | redisTemplate.opsForList()  .leftPop(K key)                  | 从左边弹出元素                                               |
|              | rpop key                                                     | redisTemplate.opsForList()  .rightPop(K key)                 | 从右边弹出元素                                               |
|              | lrem key count value                                         | redisTemplate.opsForList()  .remove(K key, long count, Object value) | 删除指定元素，•count > 0：从左到右，删除最多 count 个元素。  count < 0：从右到左，删除最多 count绝对值 个元素。  count = 0:删除所有 |
|              | ltrim key start stop                                         | redisTemplate.opsForList()  .trim(K key, long start, long  end) | 按照指定索引修剪列表                                         |
| **Set**      | sadd key element [element ...]                               | redisTemplate.opsForSet()  .add(K key, V... values)          | 添加一个或多个元素                                           |
|              | srem key element [element ...]                               | redisTemplate.opsForSet()  .remove(K key, Object... values)  | 删除一个或多个元素                                           |
|              | scard key                                                    | redisTemplate.opsForSet()  .size(K key)                      | 获取元素个数                                                 |
|              | sismember key element                                        | redisTemplate.opsForSet()  .isMember(K key, Object o)        | 判断元素是否在Set中                                          |
|              | srandmember key [count]                                      | redisTemplate.opsForSet()  .randomMember(K key)              | 随机从集合中返回指定个数的元素。默认为一个                   |
|              | spop key                                                     | redisTemplate.opsForSet()  .pop(K key)                       | 随机从集合中弹出一个元素                                     |
|              | smembers key                                                 | redisTemplate.opsForSet()  .members(K key)                   | 获取所有的元素                                               |
|              | sinter key [key ...]                                         | redisTemplate.opsForSet()  .intersect(K key, K otherKey)     redisTemplate.opsForSet()  .intersect(Collection<K> keys) | 求多个集合的交集                                             |
|              | suinon key [key ...]                                         | redisTemplate.opsForSet()  .union(K key, K otherKey)     redisTemplate.opsForSet()  .union(Collection<K> keys) | 求多个集合的并集                                             |
|              | sdiff key [key ...]                                          | redisTemplate.opsForSet()  .difference(K key, K otherKey)     redisTemplate.opsForSet()  .difference(Collection<K> keys) | 求多个集合的差集                                             |
|              | sinterstore destination key [key ...]                        | redisTemplate.opsForSet()  .intersectAndStore(Collection<K> keys,  K destKey) | 求多个集合的交集并保存                                       |
|              | suionstore destination key [key  ...]                        | redisTemplate.opsForSet()  .unionAndStore(Collection<K>  keys, K destKey) | 求多个集合的并集并保存                                       |
|              | sdiffstore destination key [key ...]                         | redisTemplate.opsForSet()  .differenceAndStore(Collection<K> keys,  K destKey) | 求多个集合的差集并保存                                       |
| **Zset**     | zadd key score member [score member  ...]                    | redisTemplate.opsForZSet()  .add(K key, V value, double  score)     redisTemplate.opsForZSet()  .add(K key, Set<TypedTuple<V>>  tuples) | 添加一个或多个带有score的元素                                |
|              | zcard key                                                    | redisTemplate.opsForZSet()  .zCard(K key)                    | 计算元素个数                                                 |
|              | zscore key member                                            | redisTemplate.opsForZSet()  .score(K key, Object o)          | 计算摸个成员的score                                          |
|              | zrank key member                                             | redisTemplate.opsForZSet()  .rank(K key, Object o)           | 从低到高计算成员的排名                                       |
|              | zrevrank key member                                          | redisTemplate.opsForZSet()  .reverseRank(K key, Object o)    | 从高到低计算成员的排名                                       |
|              | zrem key member [member ...]                                 | redisTemplate.opsForZSet()  .remove(K key, Object... values) | 删除一个或者多个成员                                         |
|              | zincrby key increment member                                 | redisTemplate.opsForZSet()  .incrementScore(K key, V value,  double delta) | 为成员的score增加increment                                   |
|              | zrange key start end [withscores]                            | redisTemplate.opsForZSet()  .range(K key, long start, long end)     redisTemplate.opsForZSet()  .rangeWithScores(K key, long start, long  end) | 从低到高返回指定排名的有序集合，withscores带有分数权值       |
|              | zrevrange key start end  [withscores]                        | redisTemplate.opsForZSet()  .reverseRange(K key, long start,  long end) | 从高到低返回score排名的有序集合，withscores带有分数权值      |
|              | zrangebyscore key min max [withscores]  [limit offset count] | redisTemplate.opsForZSet()  .reverseRangeWithScores(K key, long  start, long end) | 从低到高返回指定分数返回的成员                               |
|              | zremrangebyrank key start end                                | redisTemplate.opsForZSet()  .removeRange(K key, long start,  long end) | 删除指定排名内的升序元素                                     |
|              | zremrangebyscore key min max                                 | redisTemplate.opsForZSet()  .removeRangeByScore(K key, double min,  double max) | 删除指定score范围内的成员                                    |
|              | zinterstore destination numkeys  key [key ...] [weights weight [weight ...]]  [aggregate sum\|min\|max] | redisTemplate.opsForZSet()  .intersectAndStore(K key, K  otherKey, K destKey)     redisTemplate.opsForZSet()  .intersectAndStore(K key, Collection<K>  otherKeys, K destKey, Aggregate aggregate, Weights weights) | 指定多个集合进行交集运算，destination代表存储的目标集合，numkeys代表进行交集运算的集合个数，aggregate指定交集运算后的score的计算方式，默认是相加 |
|              | zunionstore destination numkeys key [key  ...] [weights weight [weight ...]] [aggregate sum\|min\|max] | redisTemplate.opsForZSet()  .unionAndStore(K key, K otherKey, K  destKey)     redisTemplate.opsForZSet()  .unionAndStore(K key, Collection<K>  otherKeys, K destKey, Aggregate aggregate, Weights weights) | 指定多个集合进行并集运算，规则同交集运算                     |

 