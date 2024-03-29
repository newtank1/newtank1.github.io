---
layout: post
title: Springboot中Cache相关注解
date: 2022-09-25
tags: Spring
---

在开发中，数据库与缓存是十分常见的概念。而其中，Redis常常作为缓存。

### 为什么使用Redis作为缓存

相比于大部分将数据保存在硬盘的数据库，Redis的存取都是在内存中完成的（也有和硬盘的交互，但其次数少），对内存的存取速度往往比硬盘高几个数量级。同时，Redis的KV存储结构也适合缓存的快速搜索。基于这些原因，Redis成为了开发中常用的缓存数据库。

### Springboot中集成Redis缓存

Springboot中提供了快速集成Redis的方法。

引入依赖：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

即可使用Redis相关api。

在进行配置与Redis的连接：

```yaml
spring:
  redis:
    host: localhost
    timeout: 18000000
    port: 6379
    lettuce:
      pool:
        max-active: -1
        max-wait: -1
        max-idle: 10
        min-idle: 0
```

由于在业务代码中主动去读写Redis会产生大量冗余代码、加大程序员的心智负担，Springboot提供了一系列类与注解来简化缓存操作。

我们可以进行Bean配置来管理缓存。

```java
@Bean
    @SuppressWarnings("all")
    public CacheManager cacheManager(RedisConnectionFactory lettuceConnectionFactory) {
        RedisCacheConfiguration defaultCacheConfig = RedisCacheConfiguration.defaultCacheConfig();
        defaultCacheConfig = defaultCacheConfig.entryTtl(Duration.ofSeconds(DEFAULT_EXPIRE_TIME))
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .disableCachingNullValues();

        Set<String> cacheNames = new HashSet<>();
        cacheNames.add(USER_CACHE_NAME);

        Map<String, RedisCacheConfiguration> configMap = new HashMap<>();
        configMap.put(USER_CACHE_NAME, defaultCacheConfig.entryTtl(Duration.ofSeconds(USER_CACHE_EXPIRE_TIME)));

        RedisCacheManager cacheManager = RedisCacheManager.builder(lettuceConnectionFactory)
                .cacheDefaults(defaultCacheConfig)
                .initialCacheNames(cacheNames)
                .withInitialCacheConfigurations(configMap)
                .build();
        return cacheManager;
    }
```

这个Bean提供了一个缓存管理器，它可以自动帮我们进行Redis的缓存读写操作。

### @Cache相关注解的使用

Springboot为我们提供了一系列注解来为方法与类进行缓存自动操作。要使这些注解生效，我们只需要在包中任意一个类上添加上@EnableCaching注解即可：

```java
@RestController
@EnableCaching
public class DummyController {
}
```

这个注解可以放在任意一个类上，无论是启动类、配置类还是某个Bean，都是可以激活缓存功能的。

接下来就是进行具体的缓存操作了。

#### @Cacheable

对于某个方法，如果我们想对这个方法的执行结果进行缓存，我们只需为其加上@Cacheable注解：

```java
@Cacheable(cacheNames = "DemoService-cache",keyGenerator = "myKeyGenerator")
    public String hello(String user){
        System.out.println("Exec hello!");
        return "hello "+user+random.nextInt()+"!";
    }
```

随后在调用这个方法时，Springboot生成的代理会首先检查缓存数据库（不一定是Redis，但我们提供了Redis相关配置后选择了Redis）中是否有相关缓存以及是否满足缓存条件，如果有，则不执行这个方法，直接返回缓存中的内容，否则执行方法并更新缓存。缓存的Key的格式为"{cacheName}::{key}"，这两者通过参数指定

这个注解包含若干参数：

- cacheNames：缓存的“域”，其是一个字符串数组。生成缓存时，会为每一个{cacheName}生成一个缓存key。例如数组内容为{"name1","name2"}时，会对应两个key"name1::{key}"和"name2::{key}"。只要其中一个存在，那么就是缓存命中。当两个都不存在时，方法会创建这两个key。
- key：缓存的key，使用spEL表达式编写，如果为空则根据参数列表定义
- keyGenerator：缓存的key生成器，和key参数不能同时出现，作用会在后面提到。
- condition：触发缓存的条件，使用spEL表达式编写。

#### @CachePut

使用这个注解，该方法无论缓存是否命中都会执行，并且更新缓存。

该注解也可以标记类，含义为为所有方法加上该注解。

这个注解的参数和@Cacheable是一样的。

#### @CacheEvict

这个注解标记的方法会清除相关的缓存

```java
    @CacheEvict(cacheNames = "DemoService-cache",keyGenerator = "myKeyGenerator")
    public String delete(String user){
        System.out.println("Exec delete!");
        return mapper.delete(user);
    }
```

在调用这个方法后，"{cacheName}::{key}"对应的键值对会被删除。

这个注解除了和上面两个注解都有的参数外，还有的的参数：

- allEntries：使用spEL表达式编写，执行时如果为true则清除cacheNames空间下的全部键值对，否则只清除和key相关的键值对。
- beforeInvocation：为true则在程序执行前清空缓存，否则在程序执行后清空缓存。

#### @Caching

如果一个方法同时包含以上三个注解中的多个功能，则使用该注解来组合，例如

```java
@Caching(
	  cacheable = {
        @Cacheable(cacheNames = "DemoService-cache",keyGenerator = "myKeyGenerator")
    },put = {
        @CachePut(cacheNames = "DemoService-cache1",keyGenerator = "myKeyGenerator")
    },evict = {
        @CacheEvict(cacheNames = "DemoService-cache2",keyGenerator = "myKeyGenerator"),
        @CacheEvict(cacheNames = "DemoService-cache3",keyGenerator = "myKeyGenerator")
    })
```

，就同时组合了cacheable、put、evict三个注解。

### KeyGenerator

有时方法的参数非常复杂，我们需要定制的缓存关键字生成策略。Springboot提供了一个接口KeyGenerator来让我们自定义生成策略。

这个接口是一个函数式接口：

```java
@FunctionalInterface
public interface KeyGenerator {
    Object generate(Object target, Method method, Object... params);
}
```

我们可以自定义一个实现类：

```java
@Bean
public KeyGenerator myKeyGenerator() {
    return (o, method, params) -> {
        StringJoiner sb = new StringJoiner(",","(",")");
        for (Object param : params) {
            sb.add(param.toString());
        }
        return sb.toString();
    };
}
```

在调用缓存相关方法前，如果在注解参数中指定了keyGenerator = "myKeyGenerator"，那么就会调用这个lambda表达式来生成形如"(参数1,参数2,...)"的key名。对于复杂的参数列表，我们可以定制更精细的实现类来达到目的。