---
title: Springboot接入Redis
date: 2020-02-28 09:54:01
tags: [JAVA,Spring,Springboot]
categories: JAVA


---

SpringBoot接入Redis

<!-- more -->

### Springboot配置Redis

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



### 常用功能封装RedisUtil

```java
@Slf4j
@Component
public class RedisUtil {

    @Resource
    private StringRedisTemplate stringRedisTemplate;
  
	  private ThreadLocal<String> lockLocal = new ThreadLocal<>();

    private RedisScript<Long> releaseLockScript;
  
	  private static final String RELEASE_LOCK_LUA_SCRIPT = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
  
  	@PostConstruct
    public void init() {
        // 设置解锁lua脚本
        releaseLockScript = new DefaultRedisScript<>(RELEASE_LOCK_LUA_SCRIPT, Long.class);
    }

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
  
  	public Long ttl(String key) {
        return stringRedisTemplate.getExpire(key);
    }

    public Boolean lock(String key, long timeout) {
        return lock(key, GLOBAL_LOCK_VALUE, timeout, TimeUnit.MILLISECONDS);
    }

    public Boolean lock(String key, String value, long timeout, TimeUnit timeUnit) {
        return stringRedisTemplate.opsForValue().setIfAbsent(key, value, timeout, timeUnit);
    }

    public Boolean expire(String key, long timeout, TimeUnit timeUnit) {
        return stringRedisTemplate.expire(key, timeout, timeUnit);
    }
  
  	public String getSequenceNum() {
        //14位当前时间
        String currentDateString = DateUtils.dateToString(new Date(), 8);
        Long value;
        try {
            value = this.increment(ID_NUM_KEY_PREFIX + currentDateString);
            this.expire(ID_NUM_KEY_PREFIX + currentDateString, 5, TimeUnit.SECONDS);
        } catch (Exception e) {
            log.error("redis 请求失败!", e);
            value = Long.valueOf(RandomStringUtils.randomNumeric(6));
        }
        int restSeat = NO_LIMIT - currentDateString.length();
        return currentDateString + StringUtils.leftPad(String.valueOf(value), restSeat, '0');
    }

    public String getIdNum(BusinessNoPrefixEnum prefixEnum) {
        return prefixEnum.getPrefix() + getIdNum();
    }


    public Long increment(@NotEmpty String key) {
        Long incr = stringRedisTemplate.opsForValue().increment(key);
        return incr;
    }
  
  public boolean getSpinLock(String key, long timeout, int retryTimes, long sleepTime) {
        AtomicInteger count = new AtomicInteger();
        String uuid = UUID.randomUUID().toString();

        while (!lock(key, uuid, timeout, TimeUnit.MILLISECONDS)) {
            if (retryTimes == count.incrementAndGet()) {
                log.warn("获取分布式锁失败，获取次数达到限制。 threadName:{}, key:{}, count:{}, retryTimes:{}", Thread.currentThread().getName(), key, count, retryTimes);
                return false;
            }
            try {
                log.warn("获取分布式锁失败，等待自旋获取。 threadName:{}, key:{}, count:{}, retryTimes:{}", Thread.currentThread().getName(), key, count, retryTimes);
                Thread.sleep(sleepTime);
            } catch (InterruptedException e) {
                log.error("线程等待出现异常", e);
            }
        }
        lockLocal.set(uuid);
        log.info("获取分布式锁成功。 threadName:{}, key:{}, count:{}, retryTimes:{}", Thread.currentThread().getName(), key, count, retryTimes);
        return true;
    }

    public void unlockSpinLock(String key) {

        // 从ThreadLocal取到加锁的uuid
        String uuid = lockLocal.get();
        String threadName = Thread.currentThread().getName();

        try {
            // 参数一：redisScript，参数二：key列表，参数三：arg（可多个）
            Long result = stringRedisTemplate.execute(releaseLockScript, Collections.singletonList(key), uuid);
            if (Objects.isNull(result)) {
                log.error("调用lua脚本解锁失败, key:{}, value:{}, threadName:{}", key, uuid, threadName);
                remove(key);
            }
        } catch (Exception e) {
            log.error("调用lua脚本解锁失败, key:{}, value:{}, threadName:{}, e:{}", key, uuid, threadName, e);
            remove(key);
        } finally {
            lockLocal.remove();
        }
    }

}
```

