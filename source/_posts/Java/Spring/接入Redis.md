---
title: Springboot接入Redis
date: 2020-02-28 09:54:01
tags: [JAVA,Spring,Springboot]
categories: JAVA


---

从0开始接入Redis

<!-- more -->

#### 1.Redis基础知识

Redis 可以存储键与5种不同数据结构类型之间的映射，这5种数据结构类型分别为：`String`（字符串）、`List`（列表）、`Set`（集合）、`Hash`（散列）和 `zSet`（有序集合）。

- String

**结构存储的值：**
  可以是字符串、整数或者浮点数。

**结构的读写能力：**
  对整个字符串或者字符串的其中一部分执行操作，对象和浮点数执行自增(increment)或者自减(decrement)。

- List

**结构存储的值：**
  一个链表，链表上的每个节点都包含了一个字符串。

**结构的读写能力：**
  从链表的两端推入或者弹出元素，根据偏移量(offset)对链表进行修剪(trim)，读取单个或者多个元素，根据值来查找或者移除元素。

- Set

**结构存储的值：**
  包含字符串的无序收集器(unOrderedCollection)，并且被包含的每个字符串都是独一无二的、各不相同。

**结构的读写能力：**
  添加、获取、移除单个元素，检查一个元素是否存在于某个集合中，计算交集、并集、差集，从集合里面随机获取元素。

- Hash

**结构存储的值：**
  包含键值对的无序散列表。

**结构的读写能力：**
  添加、获取、移除单个键值对，获取所有键值对。

- zSet

**结构存储的值：**
  字符串成员(member)与浮点数分值(score)之间的有序映射，元素的排列顺序由分值(score)的大小决定。

**结构的读写能力：**
  添加、获取、删除单个元素，根据分值(score)范围(range)或者成员来获取元素。

#### 2.Springboot配置Redis

如果代码需要区分不通组件的配置，可以新建一个配置，名称风格可以和spring的统一，比如`application-redis.properties`或者`application-redis.yml`，然后在spring基础配置里加上`spring.profiles.include=redis`

如不需要区分，可以直接把配置信息贴到spring基础配置文件里，下面是一些常用配置（此处配置为`Jedis`，也可以选用SpringBoot 1.5之后的`lettuce`）

```properties
#redis配置
# Redis数据库索引（默认为0，集群状态只有一个）
spring.redis.database=0
# Redis集群模式，host:port  用,隔开
#spring.redis.cluster.nodes=
# Redis服务器地址
spring.redis.host=xxx
# Redis服务器连接端口
spring.redis.port=6379
# Redis服务器连接密码（默认为空）
spring.redis.password=xxx

#以下为配置jedis链接池
# 连接池最大连接数（使用负值表示没有限制）
spring.redis.jedis.pool.max-active=200
# 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.jedis.pool.max-wait=10000
# 连接池中的最大空闲连接
spring.redis.jedis.pool.max-idle=8
# 连接池中的最小空闲连接
spring.redis.jedis.pool.min-idle=0
# 连接超时时间（毫秒）
spring.redis.timeout=10000

```

之后可以进行可选配置，比如配置@Cacheable注解、配置序列化器等，这时候可以用代码配置

```java
@Configuration
public class RedisConfig extends CachingConfigurerSupport {

    /**
     * 配置 @Cacheable 缓存注解
     * @param connectionFactory Thread-safe factory of Redis connections.
     * @return CacheManager
     */
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {

        RedisCacheWriter redisCacheWriter = RedisCacheWriter.nonLockingRedisCacheWriter(connectionFactory);
        RedisCacheConfiguration defaultCacheConfig = RedisCacheConfiguration.defaultCacheConfig();
        //设置默认超过期时间是5分钟
        defaultCacheConfig.entryTtl(Duration.ofMinutes(5));
        return new RedisCacheManager(redisCacheWriter, defaultCacheConfig);
    }

  	/**
  	 * 配置序列化器（使用Jackson，也可以用Fastjson之类的）
     */
    @Bean("redisTemplate")
    public RedisTemplate<String, Object> redisTemplateObject(RedisConnectionFactory redisConnectionFactory) {
        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(jackson2JsonRedisSerializer);
        template.setValueSerializer(jackson2JsonRedisSerializer);
        template.setHashKeySerializer(jackson2JsonRedisSerializer);
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }
  
  	/**
     * 也可以配置string操作模板，比较简单
     * @param redisConnectionFactory
     * @return
     */
  	@Bean("stringRedisTemplate")
    public StringRedisTemplate stringRedisTemplateObject(RedisConnectionFactory redisConnectionFactory) {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}
```



#### 3.简单RedisUtil

```java
@Slf4j
@Component
public class RedisUtil {

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    public String get(@NotNull String key) {
        return stringRedisTemplate.opsForValue().get(key);
    }

    public <T> T get(@NotNull String key, Class<T> t) {

        String value = stringRedisTemplate.opsForValue().get(key);

        if (StringUtils.isNoneBlank(value)) {
            return JacksonUtils.fromJson(value, t);
        }
        return null;
    }

    public <T> void set(String key, T data) {
        this.setForExpire(key, data, 2, TimeUnit.HOURS);
        log.info("save redis cache key={} done value={}", key, data);
    }

    public void set(String key, String value) {
        this.setForExpire(key, value, 2, TimeUnit.HOURS);
        log.info("save redis cache key={} done value={}", key, value);
    }

    public void setForExpire(String key, String data, long timeout, TimeUnit timeUnit) {
        stringRedisTemplate.opsForValue().set(key, data, timeout, timeUnit);
    }

    public <T> void setForExpire(String key, T data, long timeout, TimeUnit timeUnit) {
        String value = JacksonUtils.toJson(data);
        stringRedisTemplate.opsForValue().set(key, value, timeout, timeUnit);
    }

    public void remove(String key) {
        stringRedisTemplate.delete(key);
    }

    public Boolean exists(String key) {
        return stringRedisTemplate.hasKey(key);
    }

}
```

