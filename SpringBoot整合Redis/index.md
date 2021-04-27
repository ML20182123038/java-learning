---
title: SpringBoot整合Redis
description: Redis介绍，Redis序列化机制，封装Redis工具类。
date: 2021-04-26
image: https://gitee.com/ycfxhsw/picture/raw/master/ssredis.png
tags: 
    - Redis
    - SpringBoot
    - JAVA
categories: 
    - Spring
    - SpringBoot
    - JAVA
    - 数据库
---

### Redis

Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。 它支持多种类型的数据结构，如 [字符串（strings）](http://www.redis.cn/topics/data-types-intro.html#strings)， [散列（hashes）](http://www.redis.cn/topics/data-types-intro.html#hashes)， [列表（lists）](http://www.redis.cn/topics/data-types-intro.html#lists)， [集合（sets）](http://www.redis.cn/topics/data-types-intro.html#sets)， [有序集合（sorted sets）](http://www.redis.cn/topics/data-types-intro.html#sorted-sets) 与范围查询， [bitmaps](http://www.redis.cn/topics/data-types-intro.html#bitmaps)， [hyperloglogs](http://www.redis.cn/topics/data-types-intro.html#hyperloglogs) 和 [地理空间（geospatial）](http://www.redis.cn/commands/geoadd.html) 索引半径查询。 Redis 内置了 [复制（replication）](http://www.redis.cn/topics/replication.html)，[LUA脚本（Lua scripting）](http://www.redis.cn/commands/eval.html)， [LRU驱动事件（LRU eviction）](http://www.redis.cn/topics/lru-cache.html)，[事务（transactions）](http://www.redis.cn/topics/transactions.html) 和不同级别的 [磁盘持久化（persistence）](http://www.redis.cn/topics/persistence.html)， 并通过 [Redis哨兵（Sentinel）](http://www.redis.cn/topics/sentinel.html)和自动 [分区（Cluster）](http://www.redis.cn/topics/cluster-tutorial.html)提供高可用性（high availability）。

Redis 是目前业界使用最广泛的内存数据存储。相比 Memcached，Redis 支持更丰富的数据结构，例如 hashes, lists, sets 等，同时支持数据持久化。

除此之外，Redis 还提供一些类数据库的特性，比如事务，HA，主从库。可以说 Redis 兼具了缓存系统和数据库的一些特性，因此有着丰富的应用场景。本文介绍 Redis 在 Spring Boot 中两个典型的应用场景。



### Redis API 介绍

Spring Boot 提供的 Redis API 分为 **高阶 API** 和 **低阶 API**，高阶 API 是经过一定封装后的 API，而低阶 API 的使用和直接使用 Redis 的命令差不多。

高阶 API 提供了两个类可以供我们使用，分别是 `RedisTemplate` 和 `StringRedisTemplate`。StringRedisTemplate 继承自 RedisTemplate，StringRedisTemplate 的序列化方式与 RedisTemplate 的序列化的方式不同。具体在使用上的差别不是特别明显，但是数据在存储到 Redis 中的时候，因为序列化方式的不同，会有一定的差别。

低阶 API 其实也是通过 RedisTemplate 或 StringRedisTemplate 来进行获取。低阶 API 的方法和 Redis 的命令差不多。



### Redis序列化

##### 1、什么是序列化和反序列化

序列化：将对象写到IO流中

反序列化：从IO流中恢复对象

序列化的意义：序列化机制允许将实现序列化的Java对象转换位字节序列，这些字节序列可以保存在磁盘上，或通过网络传输，以达到以后恢复成原来的对象。序列化机制使得对象可以脱离程序的运行而独立存在。

##### 2、只需要配置一下，就可以解决刚才出现的问题，但是这么多序列化的手段如何挑选呢，我比较好奇，所以我又稍微深挖了一下:

```java
/**
 * Description: Redis配置
 */
@Configuration
public class MyRedisConfig {
    /**
     * redisTemplate配置
     * 序列化的几种方式:
     * OxmSerializer
     * ByteArrayRedisSerializer
     * GenericJackson2JsonRedisSerializer
     * GenericToStringSerializer
     * StringRedisSerializer
     * JdkSerializationRedisSerializer
     * Jackson2JsonRedisSerializer
     * @param redisConnectionFactory redis连接工厂
     * @return RedisTemplate
     */
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        // 配置连接工厂
        template.setConnectionFactory(redisConnectionFactory);
        // 设置key的序列化方式
        template.setKeySerializer(new StringRedisSerializer());
        // 设置value的序列化方式
        template.setValueSerializer(new Jackson2JsonRedisSerializer<Object>(Object.class));
        return template;
    }
}
```

| 名称                               | 说明                                                      |
| :--------------------------------- | --------------------------------------------------------- |
| ByteArrayRedisSerializer           | 数组序列化                                                |
| GenericJackson2JsonRedisSerializer | 使用Jackson进行序列化                                     |
| GenericToStringSerializer          | 将对象泛化成字符串并序列化，和StringRedisSerializer差不多 |
| Jackson2JsonRedisSerializer        | 使用Jackson序列化对象为json                               |
| JdkSerializationRedisSerializer    | jdk自带的序列化方式，需要实现Serializable接口             |
| OxmSerializer                      | 用xml格式存储                                             |
| StringRedisSerializer              | 简单的字符串序列化                                        |



##### 3、比较几种常见序列化手段的差异

```java
@SpringBootTest
class CacheApplicationTests {

    /**
     * 测试几种序列化手段的效率
     */
    @Test
    void test() {
        User user = new User();
        user.setId(1);
        user.setUsername("张三");
        user.setPassword("123");
        List<Object> list = new ArrayList<>();

        for (int i = 0; i < 2000; i++) {
            list.add(user);
        }

        
        // 使用GenericJackson2JsonRedisSerializer做序列化(效率太低,不推荐使用)
        GenericJackson2JsonRedisSerializer g2 = new GenericJackson2JsonRedisSerializer();
        long g2s = System.currentTimeMillis();
        byte[] byteG2 = g2.serialize(list);
        long g2l = System.currentTimeMillis();
        System.out.println("GenericJackson2JsonRedisSerializer序列化消耗的时间：" + (g2l - g2s) + "ms,序列化之后的长度：" + byteG2.length);
        g2.deserialize(byteG2);
        System.out.println("GenericJackson2JsonRedisSerializer反序列化的时间：" + (System.currentTimeMillis() - g2l) + "ms");

        // 使用GenericToStringSerializer做序列化(和StringRedisSerializer差不多,效率没有StringRedisSerializer高,不推荐使用)
        GenericToStringSerializer g = new GenericToStringSerializer(Object.class);

        long gs = System.currentTimeMillis();
        byte[] byteG = g.serialize(list.toString());
        long gl = System.currentTimeMillis();
        System.out.println("GenericToStringSerializer序列化消耗的时间：" + (gl - gs) + "ms,序列化之后的长度：" + byteG.length);
        g.deserialize(byteG);
        System.out.println("GenericToStringSerializer反序列化的时间：" + (System.currentTimeMillis() - gl) + "ms");


        // 使用Jackson2JsonRedisSerializer做序列化(效率高,适合value值的序列化)
        Jackson2JsonRedisSerializer j2 = new Jackson2JsonRedisSerializer(Object.class);
        long j2s = System.currentTimeMillis();
        byte[] byteJ2 = j2.serialize(list);
        long j2l = System.currentTimeMillis();
        System.out.println("Jackson2JsonRedisSerializer序列化消耗的时间：" + (j2l - j2s) + "ms,序列化之后的长度：" + byteJ2.length);
        j2.deserialize(byteJ2);
        System.out.println("Jackson2JsonRedisSerializer反序列化的时间：" + (System.currentTimeMillis() - j2l) + "ms");

        
        // 使用JdkSerializationRedisSerializer,实体类必须实现序列化接口(不推荐使用)
        JdkSerializationRedisSerializer j = new JdkSerializationRedisSerializer();
        long js = System.currentTimeMillis();
        byte[] byteJ = j.serialize(list);
        long jl = System.currentTimeMillis();
        System.out.println("JdkSerializationRedisSerializer序列化消耗的时间：" + (jl - js) + "ms,序列化之后的长度：" + byteJ.length);
        j.deserialize(byteJ);
        System.out.println("JdkSerializationRedisSerializer反序列化的时间：" + (System.currentTimeMillis() - jl) + "ms");


        // 使用StringRedisSerializer做序列化(效率非常的高,但是比较占空间,只能对字符串序列化,适合key值的序列化)
        StringRedisSerializer s = new StringRedisSerializer();

        long ss = System.currentTimeMillis();
        byte[] byteS = s.serialize(list.toString());
        long sl = System.currentTimeMillis();
        System.out.println("StringRedisSerializer序列化消耗的时间：" + (sl - ss) + "ms,序列化之后的长度：" + byteS.length);
        s.deserialize(byteS);
        System.out.println("StringRedisSerializer反序列化的时间：" + (System.currentTimeMillis() - sl) + "ms");


        // 使用FastJson做序列化,这个表现为什么这么差我也不是很明白
        FastJsonRedisSerializer<Object> f = new FastJsonRedisSerializer<>(Object.class);

        long fs = System.currentTimeMillis();
        byte[] byteF = f.serialize(list);
        long fl = System.currentTimeMillis();
        System.out.println("FastJsonRedisSerializer序列化消耗的时间：" + (fl - fs) + "ms,序列化之后的长度：" + byteF.length);
        f.deserialize(byteF);
        System.out.println("FastJsonRedisSerializer反序列化的时间：" + (System.currentTimeMillis() - fl) + "ms");


        // 使用FastJson(效率高,序列化后占空间也很小,推荐使用)
        GenericFastJsonRedisSerializer gf = new GenericFastJsonRedisSerializer();

        long gfs = System.currentTimeMillis();
        byte[] byteGf = gf.serialize(list);
        long gfl = System.currentTimeMillis();
        System.out.println("GenericFastJsonRedisSerializer序列化消耗的时间：" + (gfl - gfs) + "ms,序列化之后的长度：" + byteGf.length);
        gf.deserialize(byteGf);
        System.out.println("GenericFastJsonRedisSerializer反序列化的时间：" + (System.currentTimeMillis() - gfl) + "ms");
    }
}
```



##### 4、总结

| 名称                           | 序列化效率 | 反序列化效率 | 占用空间 | 是否推荐使用          |
| ------------------------------ | ---------- | ------------ | -------- | --------------------- |
| StringRedisSerializer          | 很高       | 很高         | 高       | 推荐给key进行序列化   |
| Jackson2JsonRedisSerializer    | 高         | 较高         | 偏高     | 推荐给value进行序列化 |
| GenericFastJsonRedisSerializer | 高         | 较低         | 较低     | 推荐给value进行序列化 |



##### 5、附上Redis序列化配置文件

```java
/**
 * @Description: Redis配置
 */
@Configuration
public class MyRedisConfig {
    /**
     * redisTemplate配置
     * @param redisConnectionFactory redis连接工厂
     * @return RedisTemplate
     */
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        // 配置连接工厂
        template.setConnectionFactory(redisConnectionFactory);
        // 配置key的序列化方式
        template.setKeySerializer(new StringRedisSerializer());
        // 使用Jackson2JsonRedisSerializer配置value的序列化方式
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
        // 使用FastJson配置value的序列化方式
// template.setValueSerializer(new GenericFastJsonRedisSerializer());
        return template;
    }
}
```



### 封装RedisUtils工具类

##### 引入Redis依赖

```xml
<!--SpringBoot与Redis整合依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

##### 设置Redis的Template ——> RedisConfig.java

```java
/**
 * @Description: Redis配置类
 */
@Configuration
public class RedisConfig {
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();
        RedisSerializer<String> redisSerializer = new StringRedisSerializer();
        redisTemplate.setConnectionFactory(factory);

        // key序列化
        redisTemplate.setKeySerializer(redisSerializer);
        // value序列化
        redisTemplate.setValueSerializer(redisSerializer);
        // key hashmap序列化
        redisTemplate.setHashKeySerializer(redisSerializer);
        // value hashmap序列化
        redisTemplate.setHashValueSerializer(redisSerializer);
        return redisTemplate;
    }
}
```

##### 设置连接信息

```properties
#Redis
# 连接的那个数据库
spring.redis.database=0
# redis服务的ip地址
spring.redis.host=127.0.0.1
# redis端口号
spring.redis.port=6379
# redis的密码，没设置过密码，可为空
spring.redis.password=ycfxhsw
```

##### Redis工具类

```java
/**
 * Redis 工具类
 */
@Service
public class RedisUtils {
    @Autowired
    private RedisTemplate redisTemplate;

    private static double size = Math.pow(2, 32);

    /**
     * 写入缓存
     * @param key
     * @param offset
     * @param isShow
     * @return result
     */
    public boolean setBit(String key, long offset, boolean isShow) {
        boolean result = false;
        try {
            ValueOperations<Serializable, Object> operations = redisTemplate.opsForValue();
            operations.setBit(key, offset, isShow);
            result = true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
    }

    /**
     * 写入缓存
     * @param key
     * @param offset
     * @return result
     */
    public boolean getBit(String key, long offset) {
        boolean result = false;
        try {
            ValueOperations<Serializable, Object> operations = redisTemplate.opsForValue();
            result = operations.getBit(key, offset);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
    }

    /**
     * 写入缓存
     * @param key
     * @param value
     * @return
     */
    public boolean set(final String key, Object value) {
        boolean result = false;
        try {
            ValueOperations<Serializable, Object> operations = redisTemplate.opsForValue();
            operations.set(key, value);
            result = true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
    }

    /**
     * 写入缓存设置时效时间
     * @param key
     * @param value
     * @return
     */
    public boolean set(final String key, Object value, Long expireTime) {
        boolean result = false;
        try {
            ValueOperations<Serializable, Object> operations = redisTemplate.opsForValue();
            operations.set(key, value);
            redisTemplate.expire(key, expireTime, TimeUnit.SECONDS);
            result = true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
    }

    /**
     * 批量删除对应的value
     * @param keys
     */
    public void remove(final String... keys) {
        for (String key : keys) {
            remove(key);
        }
    }

    /**
     * 删除对应的value
     * @param key
     */
    public void remove(final String key) {
        if (exists(key)) {
            redisTemplate.delete(key);
        }
    }

    /**
     * 判断缓存中是否有对应的value
     * @param key
     * @return
     */
    public boolean exists(final String key) {
        return redisTemplate.hasKey(key);
    }

    /**
     * 读取缓存
     * @param key
     * @return
     */
    public Object get(final String key) {
        Object result = null;
        ValueOperations<Serializable, Object> operations = redisTemplate.opsForValue();
        result = operations.get(key);
        return result;
    }

    /**
     * 哈希 添加
     * @param key
     * @param hashKey
     * @param value
     */
    public void hmSet(String key, Object hashKey, Object value) {
        HashOperations<String, Object, Object> hash = redisTemplate.opsForHash();
        hash.put(key, hashKey, value);
    }

    /**
     * 哈希获取数据
     * @param key
     * @param hashKey
     * @return
     */
    public Object hmGet(String key, Object hashKey) {
        HashOperations<String, Object, Object> hash = redisTemplate.opsForHash();
        return hash.get(key, hashKey);
    }

    /**
     * 列表添加
     * @param k
     * @param v
     */
    public void lPush(String k, Object v) {
        ListOperations<String, Object> list = redisTemplate.opsForList();
        list.rightPush(k, v);
    }

    /**
     * 列表获取
     * @param k
     * @param l
     * @param l1
     * @return
     */
    public List<Object> lRange(String k, long l, long l1) {
        ListOperations<String, Object> list = redisTemplate.opsForList();
        return list.range(k, l, l1);
    }

    /**
     * 集合添加
     * @param key
     * @param value
     */
    public void add(String key, Object value) {
        SetOperations<String, Object> set = redisTemplate.opsForSet();
        set.add(key, value);
    }

    /**
     * 集合获取
     * @param key
     * @return
     */
    public Set<Object> setMembers(String key) {
        SetOperations<String, Object> set = redisTemplate.opsForSet();
        return set.members(key);
    }

    /**
     * 有序集合添加
     * @param key
     * @param value
     * @param scoure
     */
    public void zAdd(String key, Object value, double scoure) {
        ZSetOperations<String, Object> zset = redisTemplate.opsForZSet();
        zset.add(key, value, scoure);
    }

    /**
     * 有序集合获取
     * @param key
     * @param scoure
     * @param scoure1
     * @return
     */
    public Set<Object> rangeByScore(String key, double scoure, double scoure1) {
        ZSetOperations<String, Object> zset = redisTemplate.opsForZSet();
        redisTemplate.opsForValue();
        return zset.rangeByScore(key, scoure, scoure1);
    }


    /**
     * 第一次加载的时候将数据加载到 redis 中
     * @param name
     */
    public void saveDataToRedis(String name) {
        double index = Math.abs(name.hashCode() % size);
        long indexLong = new Double(index).longValue();
        boolean availableUsers = setBit("availableUsers", indexLong, true);
    }

    /**
     * 第一次加载的时候将数据加载到redis中
     * @param name
     * @return
     */
    public boolean getDataToRedis(String name) {
        double index = Math.abs(name.hashCode() % size);
        long indexLong = new Double(index).longValue();
        return getBit("availableUsers", indexLong);
    }

    /**
     * 有序集合获取排名
     * @param key   集合名称
     * @param value 值
     */
    public Long zRank(String key, Object value) {
        ZSetOperations<String, Object> zset = redisTemplate.opsForZSet();
        return zset.rank(key, value);
    }


    /**
     * 有序集合获取排名
     * @param key
     */
    public Set<ZSetOperations.TypedTuple<Object>> zRankWithScore(String key, long start, long end) {
        ZSetOperations<String, Object> zset = redisTemplate.opsForZSet();
        Set<ZSetOperations.TypedTuple<Object>> ret = zset.rangeWithScores(key, start, end);
        return ret;
    }

    /**
     * 有序集合添加
     * @param key
     * @param value
     */
    public Double zSetScore(String key, Object value) {
        ZSetOperations<String, Object> zset = redisTemplate.opsForZSet();
        return zset.score(key, value);
    }


    /**
     * 有序集合添加分数
     * @param key
     * @param value
     * @param scoure
     */
    public void incrementScore(String key, Object value, double scoure) {
        ZSetOperations<String, Object> zset = redisTemplate.opsForZSet();
        zset.incrementScore(key, value, scoure);
    }


    /**
     * 有序集合获取排名
     * @param key
     */
    public Set<ZSetOperations.TypedTuple<Object>> reverseZRankWithScore(String key, long start, long end) {
        ZSetOperations<String, Object> zset = redisTemplate.opsForZSet();
        Set<ZSetOperations.TypedTuple<Object>> ret = zset.reverseRangeByScoreWithScores(key, start, end);
        return ret;
    }

    /**
     * 有序集合获取排名
     * @param key
     */
    public Set<ZSetOperations.TypedTuple<Object>> reverseZRankWithRank(String key, long start, long end) {
        ZSetOperations<String, Object> zset = redisTemplate.opsForZSet();
        Set<ZSetOperations.TypedTuple<Object>> ret = zset.reverseRangeWithScores(key, start, end);
        return ret;
    }
}
```

##### 测试

```java
@RestController
public class RedisController {
    @Autowired
    private RedisUtils redisUtils;

    @RequestMapping("setAndGet")
    public String test(String k, String v) {
        redisUtils.set(k, v);
        return (String) redisUtils.get(k);
    }
}
```

