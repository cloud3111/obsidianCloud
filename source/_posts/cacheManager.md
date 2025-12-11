---
banner: "[[pixel-banner-image.png]]"
title: 了解缓存的各种表现以及实现方法
tags:
  - Cache
  - Redis
categories: 编程
date: 2025-04-15T21:12:00
---

	No matter what happens, I’ve got your back.
# 缓存处理提高并发速率
	在高并发系统中，缓存是提升性能、减轻数据库压力的关键手段。本文从数据库缓存、前端缓存、SpringCache 多种缓存管理器等角度，系统性地梳理了缓存机制与实践经验。
---
## 数据库的内部缓存
### 引擎缓存池(InnoDB Buffer Pool)
- 可以缓存**磁盘上经常操作的真实数据**，在执行增删改查操作时，先操作缓冲池中的数据（若缓冲池没有数据，则从磁盘加载并缓存），然后再以一定频率刷新到磁盘，从而减少磁盘IO，加快处理速度。
#### 存储空间

| 缓存内容             | 说明   |
| :--------------- | :--- |
| 数据页（Data Pages）  | 表的数据 |
| 索引页（Index Pages） | 索引结构 |
| undo log、插入缓冲等   | 内部使用 |
#### 优点
- 不需要手动写逻辑，MySQL 自动将**热点数据**缓存进内存
- 读操作速度快, **缓存在缓存池内存中**，读取数据就像从 `RAM` 里拿，速度比磁盘快几个数量级
- 缓存的页与磁盘数据由 `InnoDB` 自己维护，不存在“缓存不一致”的问题
- 不仅是数据页，索引结构也会缓存在内存中，提高复杂查询的执行效率
#### 缺点
- `InnoDB` 缓存是**本地内存缓存**，**不能跨服务、跨节点共享，不适合分布式环境**
- **无法像 Redis 那样对特定 key 设置 TTL、失效策略, 不可自定义**
- 没有 `@Cacheable` 那种“缓存先读，数据库兜底”的能力
- 缓存命中率低, 热点业务不会优先
### `Mybatis` 缓存
#### 一级缓存
- 同一个事务前提下, 保存重复的sql语句执行结果, **默认在一个会话线程有效**, 默认开启
- 生效条件: **事务 中途不发生增删改(发生就清空缓存)**
```java
@Transactional
public void testCache() {
    userMapper.selectById(1L); // 第一次查
    userMapper.selectById(1L); // 第二次查，命中缓存（只要 Spring 管理的是同一个会话线程）
}
```
#### 二级缓存
-  **在xml文件中使用<cache />开启, 不同会话下保存sql执行结果保存到本地缓存
- 当前 xxxMapper.xml范围有效,  nameSpace中方法为key, 缓存数据为value
- 提高查询效率，减少数据库访问次数, 缓存命中率: Cache HitRatio 0.5
- 生效条件: **手动开启 只能在xml文件中定义 中途不发生增删改(发生就清空数据) 实体类需要序列化**
```xml
<mapper namespace="com.example.mapper.UserMapper">
  <cache /> <!-- 开启二级缓存 -->
  <select>缓存内容</select>
</mapper>
```
## 前端缓存
| 缓存方式             | 生命周期 | 特点           |
| ---------------- | ---- | ------------ |
| `sessionStorage` | 会话级  | 页面关闭即清空缓存    |
| `localStorage`   | 持久级  | 缓存持久存在，需手动清除 |
## SpringCache 缓存管理器体系

| 基本注解        | 说明                          |
| ----------- | --------------------------- |
| @Cacheable  | **缓存方法返回值**,当方法执行时有缓存直接返回缓存 |
| @CachePut   | 更新缓存内容                      |
| @CacheEvict | 删除缓存内容, 可以一次性指定前缀下的所有键值对    |
| @Caching    | 支持多个注解组合使用                  |

| 参数名                | 作用说明                                                                                                      |
| ------------------ | --------------------------------------------------------------------------------------------------------- |
| cacheManager       | 指定使用哪个**缓存管理器**（如本地或者Redis）你可以在配置中自定义多个管理器。                                                               |
| value / cacheNames | 缓存的名字，对应配置的缓存区域名，可以是 **Redis 中的前缀**。二者等价，推荐使用 `value`。                                                    |
| key                | 缓存的 key，支持 SpEL 表达式，如 `#id`、`#user.name`。默认使用方法所有参数作为 key。                                                |
| condition          | 缓存的“执行前”判断，只有满足条件才执行缓存操作。支持 `SpEL`。                                                                       |
| unless             | 缓存的“执行后”判断，如果为 true，则**不缓存**返回结果。支持 SpEL。                                                                 |
| keyGenerator       | 如果你想自定义缓存 key 生成规则（而不是 key），可以配置 key 生成器的 bean 名称。                                                        |
| sync               | 避免缓存击穿热点key问题，设置为 `true` 时，当多个线程访问同一个 key 时，只有一个线程去执行方法，其它线程阻塞等待结果（只有 RedisCacheManager 支持这个参数, **加锁行为**! |
| cacheResolve       | 替代 `cacheManager` 来决定用哪个缓存,需要配置类进行策略行为@Cacheable(cacheResolver = "myCacheResolver")                       |

### 本地缓存
#### SimpleCacheManager(默认)
- Spring Cache 默认实现 = `SimpleCacheManager` + **`ConcurrentMap`**（**基于内存**的Map缓存）
- 这个 `ConcurrentMap` 就是一个普通的线程安全 `Map`，存放在应用的 JVM 堆内存中
- 无法设置过期策略，**永远不会失效**, 所以可能需要一个定时任务来检查是否需要手动删除缓存
- 不能设置最大容量，**风险是 `OOM`（内存溢出）**
#### CaffeineManager
- Caffeine 是一个 **高性能的本地缓存库**，是 Java 里速度最快、最强大的缓存框架之一，用来替代传统的 简单的 `Map` 缓存，非常适合对性能要求高的单体项目
- **使用`ConcurrentLinkedHashMap`作为容器**
- 使用分段 `CAS` + `LRU` 策略
- 支持配置 `expireAfterWrite`, `maximumSize` 等
- 功能强大, 支持异步刷新、统计分析
- 支持大多数的缓存过期策略
![17447245005311744724500093.png|700x355](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17447245005311744724500093.png)

### 分布式缓存(多节点)
#### RedisManger
	在微服务架构中推荐使用分布式缓存实现缓存共享、状态同步。
- 支持 TTL、自定义序列化、跨服务访问
- 适用于高并发、高可用系统
- 但需要额外部署 Redis 服务，复杂度略高
## 多级缓存（Multi-Level Cache）
- 将 **本地缓存（如 Caffeine）+ 分布式缓存（如 Redis）** 结合，先查本地未命中再查 Redis，兼顾**速度与一致性**。
```java
@Cached(name="userCache-", key="#userId", expire = 3600, cacheType = CacheType.BOTH)
public User getUserById(Long userId) {
    return userRepository.findById(userId);
}
```
### 例子
#### 目标类上的注解
```java
@Cacheable(cacheManager = "redisManager",  
        value = "user",  
        key = "pageQuery + '_' + #userDTO.id"
        condition = "#userDTO.id != null ",  
        unless = "#result == null || #result.total == 0"  // 查询结果为空就不缓存  
)
```
#### 配置类
```java
/**  
 * 不同缓存管理器的配置类Manager + caffeine基本配置  
 * @author cloud_3111  
 * @since 2025-04-16  
 */@Configuration(enforceUniqueMethods = false)  
public class cacheConfig {  
  
    /**  
     * SpringCache默认使用ConcurrentHashMap作为缓存Map  
     * @return {@code CacheManager }  
     */  
    @Bean(name = "mapManager")  
    public CacheManager SimpleCacheManager() {  
        // 创建一个简单缓存管理器  
        SimpleCacheManager manager = new SimpleCacheManager();  
  
        List<ConcurrentMapCache> caches = new ArrayList<>();  
        caches.add(new ConcurrentMapCache("default"));  
        caches.add(new ConcurrentMapCache("userCache"));  
        caches.add(new ConcurrentMapCache("productCache"));  
  
        manager.setCaches(caches);  
        return manager;  
    }  
  
    /**  
     * 虽然跟ConcurrentHashMap一样是本地缓存, 但是caffeine框架支持自定义过期时间,大小…  
     * 使用concurrentLinkedHashMap作为容器,采用segment分段cas来适应高并发访问,支持异步处理(需开启)  
     * @return {@code CacheManager }  
     */  
    @Bean(name = "caffeineManager")  
    public CacheManager caffeineManager() {  
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();  
  
        // 配置 Caffeine 缓存  
        cacheManager.setCaffeine(Caffeine.newBuilder()  
                .expireAfterWrite(10, TimeUnit.MINUTES) // 设置缓存过期时间  
                .maximumSize(100) // 设置缓存最大条目  
                .weakKeys() // 使用弱引用来保存缓存的键,可被回收  
                .recordStats()); // 启用统计信息  
  
        return cacheManager;  
    }  
  
    /**  
     * 使用redis做分布式缓存  
     * 因为Jackson 在进行 Redis 缓存序列化时，无法处理 Java 8 的 LocalTime LocalDatetime… 类型和反序列的问题,需要注册转换器模块  
     * @param connectionFactory 连接工厂  
     * @return {@code CacheManager }  
     */  
    @Primary  
    @Bean(name = "redisManager")  
    public CacheManager redisManager(RedisConnectionFactory connectionFactory) {  
        // 1. 自定义 ObjectMapper       
         ObjectMapper objectMapper = new ObjectMapper();  
         // 支持 LocalDate、LocalDateTime、LocalTime  
        objectMapper.registerModule(new JavaTimeModule()); 
        // 不以时间戳形式序列化  
        objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS); 
        //让 Jackson 在序列化时加上类型信息（如 "@class": "com.xxx.pageVO"），这样反序列化才不会成 LinkedHashMap        
        objectMapper.activateDefaultTyping(  
                LaissezFaireSubTypeValidator.instance,  
                ObjectMapper.DefaultTyping.NON_FINAL,  
                JsonTypeInfo.As.PROPERTY  
        );  
  
        // 2. 创建 JSON 序列化器  
        GenericJackson2JsonRedisSerializer jackson2JsonRedisSerializer =  
                new GenericJackson2JsonRedisSerializer(objectMapper);  
  
        // 3.创建默认的序列化配置  
        RedisCacheConfiguration cacheConfig = RedisCacheConfiguration.defaultCacheConfig()  
                // 设置默认缓存有效期为10分钟  
                .entryTtl(Duration.ofMinutes(10))  
                // Key序列化  
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))  
                // Value序列化  
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer))  
                //  配置 key 前缀: 默认允许前缀和空值  
                .prefixCacheNameWith("train:business:");  
  
        // 4.创建RedisCacheManager  
        return RedisCacheManager.builder(connectionFactory)  
                .cacheDefaults(cacheConfig)  
                .build();  
    }  
  
    /**  
     * caffeine框架自带的包(不与SpringCache关联的用法)  
     * 更底层、更灵活，适合手动操作缓存：  
     * 使用方法: 也是操作底层map  
     * cache.put("key", value);  
     * cache.getIfPresent("key");  
     * cache.invalidate("key");  
     */    @Bean(name = "caffeine")  
    public Cache<Object, Object> caffeine() {  
        return Caffeine.newBuilder()  
                // 设置最后一次写入或访问后经过固定时间过期  
                .expireAfterWrite(30, TimeUnit.SECONDS)  
                // 初始的缓存空间大小  
                .initialCapacity(100)  
                // 缓存的最大条数  
                .maximumSize(1000)  
                .build();  
    }  
}
```
#### 验证CaffeineManager的正常工作
```java
@SpringBootTest(classes = businessApplication.class, properties = "spring.config.location=classpath:/application-test.yaml")  
//@EnableAutoConfiguration(exclude = {DataSourceAutoConfiguration.class})  
public class businessTest {  
  
    @Qualifier("caffeineManager")  
    @Autowired  
    private CacheManager caffeineManager;  
    
	@Qualifier("SimpleCacheManager")  
	@Autowired  
	private CacheManager SimpleCacheManager;
  
    @Autowired  
    private DailyTrainService dailyTrainService;  
  
    @Test  
    public void testCaffeineManager() {  
        DailyTrainQuery dailyTrainQuery = new DailyTrainQuery();  
        dailyTrainQuery.setTrainCode("D3306");  
        dailyTrainQuery.setPageNum(1);  
        dailyTrainQuery.setPageSize(5);  
		// 第一次查询数据库,第二次查询缓存
        dailyTrainService.queryList(dailyTrainQuery);  
        dailyTrainService.queryList(dailyTrainQuery);  
        for (String cacheName : caffeineManager.getCacheNames()) {  
            Cache cache = caffeineManager.getCache(cacheName);  
            if (cache instanceof CaffeineCache caffeineCache) {  
	            // 底层map容器: ConcurrentLinkedHashMap
                ConcurrentMap<Object, Object> map =     caffeineCache.getNativeCache().asMap();  
                System.out.println("打印缓存map开始");  
                map.forEach((key, value) -> System.out.println(key + " : " + value));  
                System.out.println("打印缓存map结束");  
                break;  
            }  
            System.out.println("未找到");  
        }  
    }  
    @Test  
	public void testSimpleCacheManager() {  
	    DailyTrainCarriageQuery dailyTrainCarriageQuery = new DailyTrainCarriageQuery();  
	    dailyTrainCarriageQuery.setTrainCode("D3307");  
	    dailyTrainCarriageQuery.setPageNum(1);  
	    dailyTrainCarriageQuery.setPageSize(5);  
	  
	    dailyTrainCarriageService.queryList(dailyTrainCarriageQuery);  
	    dailyTrainCarriageService.queryList(dailyTrainCarriageQuery);  
	    Cache cache = SimpleCacheManager.getCache("DailyTrainCarriage");  
	    System.out.println(cache.getNativeCache());  
	}
}
```
### 总结
| 类型          | 特性                    | 场景推荐      |
| ----------- | --------------------- | --------- |
| InnoDB 缓存   | 引擎自动缓存，无需配置           | 基础数据库访问优化 |
| MyBatis 缓存  | 降低 SQL 重复执行，需小心失效     | 简单系统      |
| 本地缓存        | 快速访问，单机系统最佳           | 单体应用      |
| Caffeine 缓存 | 高性能本地缓存，支持 TTL等, 功能丰富 | 高频操作本地数据  |
| Redis 缓存    | 分布式缓存，支持共享与持久化        | 微服务/多节点系统 |
- 所有的缓存管理器下的缓存管理组成
![17448769185421744876917946.png|519x437](https://fastly.jsdelivr.net/gh/cloud3111/cloudWallpaper@main/17448769185421744876917946.png)
## 缓存过期策略