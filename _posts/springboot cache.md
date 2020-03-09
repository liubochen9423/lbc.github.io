1.需要在启动类上注解@EnableCaching
2.配置CacheManager
3.控制器上注解使用@Cacheable
```java
package com.youzan.test.tata.cache;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.caffeine.CaffeineCache;
import org.springframework.cache.support.SimpleCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.ArrayList;
import java.util.List;

@Configuration
@EnableCaching(proxyTargetClass = true)
public class CacheConfiguration {

    @Bean
    public CacheManager cacheManager() {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        List<CaffeineCache> caches = new ArrayList<>();
        for (Caches c : Caches.values()) {
            caches.add(buildCache(c));
        }
        cacheManager.setCaches(caches);

        return cacheManager;
    }

    //这里的Caches为枚举定义
    private CaffeineCache buildCache(Caches c) {
        Cache<Object, Object> cache = Caffeine.newBuilder()
                .recordStats()
                .expireAfterWrite(c.getExpire(), c.getTimeUnit())
                .maximumSize(c.getMaximumSize())
                .build();
        return new CaffeineCache(c.name(), cache);
    }
}
```

接口上定义@Cacheable(c.name())