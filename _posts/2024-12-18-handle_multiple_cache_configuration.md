---
title: Handle multiple cache configurations in SpringBoot
categories: [backend]
date: 2024-12-18 10:00:00
tags: [springboot, java, cache]
image: "/assets/img/handle_multiple_cache_configuration/poster.jpeg"
---
## Context
Despite not being tied to any particular cache implementation, Spring Framework offers an abstraction for typical caching cases. However, defining specific configurations such as maximum size, expiration time (TTL), etc is not a part of this abstraction.

Let’s see the example below:

### 1) CacheConfiguration.java


```java
@Configuration
@EnableCaching
public class CacheConfiguration {
}
```


### 2) ServiceA.java

```java
class ServiceA {
   
    @Cacheable("stateCache")
    public State findState() {
       //do something
    }
 
    @Cacheable("jwksCache")
    public JWKS findJWKS() {
        //do something
    }
 
}
```

### 3) application.properties
```properties
spring.cache.type=CAFFEINE
spring.cache.cache-names=stateCache, jwksCache
spring.cache.caffeine.spec=maximumSize=5000,expireAfterWrite=10m
```

In this example, we’re using the Caffeine library to cache for two different types: state and jwks. The maximum size is 5000 items and each item will be expired after 10 minutes.

Requirement: If we want to custom the expiration time for the JWKS cache up to 60 minutes but keep the State cache still in 10 minutes. Obviously, we can’t update the application.properties because it will affect the State cache as well.

How could we deal with this case? 

##  Solution

### 1. Customize the CacheConfiguration.java

With this step, we're creating two different CaffeinieCache objects with their own configuration such as maximum size, TTL

```java
@Configuration
@EnableCaching
@AllArgsConstructor
public class CacheConfiguration {

    CacheProperties cacheProperties;

    @Bean
    public CacheManager cacheManager() {
        //Create the CaffeineCache for State cache with specific configuration
        CaffeineCache stateCache = buildCache(cacheProperties.getStateCacheName(),
                cacheProperties.getStateCacheTTL(), cacheProperties.getStateMaximumSize());
        //Create the CaffeineCache for JWKS cache with specific configuration
        CaffeineCache jwksTokenVerifyCache = buildCache(cacheProperties.getJwksTokenVerifyCacheName(),
                cacheProperties.getJwksTokenVerifyCacheTTL(), cacheProperties.getJwksMaximumSize());
        //Add two caches into the cache manager
        SimpleCacheManager manager = new SimpleCacheManager();
        manager.setCaches(Arrays.asList(stateCache, jwksTokenVerifyCache));
        return manager;
    }

    private CaffeineCache buildCache(String name, int minutesToExpire, int maximumSize) {
        return new CaffeineCache(name, Caffeine.newBuilder()
                .expireAfterWrite(minutesToExpire, TimeUnit.MINUTES)
                .maximumSize(maximumSize)
                .build());
    }

}
```

### 2) Define CacheProperties.java
```java
@Data
@Configuration
@ConfigurationProperties(prefix = "cache")
public class CacheProperties {

    private String stateCacheName;

    private int stateMaximumSize;

    private int stateCacheTTL;

    private String jwksTokenVerifyCacheName;

    private int jwksMaximumSize;

    private int jwksTokenVerifyCacheTTL;

}
```
### 3) Define configuration for each type of cache in application.properties

Now, we switch to using the customized cache configuration with the prefix “cache”. Therefore, we safely delete the spring.cache.cache-names and spring.cache.caffeine.spec which is no longer used.

```properties
spring.cache.type=CAFFEINE

#spring.cache.cache-names=stateCache, tokenVerifyES256PublicKey
#spring.cache.caffeine.spec=maximumSize=5000,expireAfterWrite=10m

cache.stateMaximumSize=5000
cache.stateCacheName=stateCache
cache.stateCacheTTL=10
cache.jwksTokenVerifyCacheName=jwksTokenVerifyCache
cache.jwksMaximumSize=100
cache.jwksTokenVerifyCacheTTL=60
```

## Summary
Although we can't customize the cache configuration on the abstraction layer. However, it's possible to create multiple cache objects that hold their own configuration and then add them back to the cache manager to catch the goal.

Enjoy coding and hope it helps.