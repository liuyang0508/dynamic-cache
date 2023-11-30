# dynamic-cache
动态缓存

## DynamicCacheAspect切面

![动态缓存](D:%5CInformation%5CPredictive%20Prevention%5Cliuyang%5CRequirement%20&%20Data%5C%E5%85%89%E7%BC%86%E6%95%B0%E5%AD%97%E5%8C%96%5C%E5%9B%BE%E6%9F%A5%E8%AF%A2%5C%E5%8A%A8%E6%80%81%E7%BC%93%E5%AD%98%5C%E5%8A%A8%E6%80%81%E7%BC%93%E5%AD%98.jpg)

## 介绍
动态缓存切面是一个基于AspectJ和Guava Cache实现的缓存策略。它可以根据缓存命中率、堆内存使用情况等因素，动态地决定是否将方法的结果缓存起来，从而提高系统性能和响应速度。

## 作用
动态缓存切面的作用是在方法执行前判断缓存中是否存在结果，并根据缓存命中率和堆内存使用情况来决定是否使用缓存。通过减少重复计算和数据库访问，可以大大提高系统性能和响应速度。

## 效果
使用动态缓存切面可以带来以下效果：

减少重复计算：对于相同的输入参数，只需计算一次，后续直接从缓存中获取结果。
减少数据库访问：对于频繁访问的数据，可以将结果缓存起来，减少对数据库的访问次数。
提高系统性能：通过减少计算和数据库访问，可以大大提高系统的性能和响应速度。

## 价值
动态缓存切面的价值在于：

提高系统性能和响应速度，提升用户体验。
减少对数据库的访问，降低数据库负载。
通过缓存策略的灵活配置，可以根据具体需求进行优化和调整。

## 实现方式
动态缓存切面的实现方式包括以下几个关键点：

- 使用AspectJ实现切面：通过在方法执行前判断缓存中是否存在结果，并根据缓存命中率和堆内存使用情况来决定是否使用缓存。
- 使用Guava Cache作为缓存实现：通过Guava Cache来存储缓存数据，并设置缓存过期时间、最大缓存数目等参数。
- 使用ConcurrentHashMap来维护计数器和上一次请求时间戳：用于判断请求是否为重复请求，并根据阈值决定是否将请求纳入缓存。

## 使用操作说明
1. 在需要使用动态缓存的方法上添加@DynamicCache注解，并配置相应的参数，如缓存过期时间、最大缓存数目等。
2. 在Spring配置文件中配置动态缓存切面的Bean，如<bean id="dynamicCacheAspect" class="com.base.common.aop.DynamicCacheAspect"/>。
3. 在需要使用动态缓存的方法上添加@DynamicCache注解，并配置相应的参数，如缓存过期时间、最大缓存数目等。
4. 运行程序，动态缓存切面

## 使用操作样例
1. 引入依赖
~~~~xml
<dependency>
    <groupId>com.base</groupId>
    <artifactId>common</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
~~~~

2. 注入动态缓存切面的Bean
~~~~java
/**
 * description: GraphQueryService 启动类
 *
 * @author l30041564
 * @since 2023-11-11
 */
@EnableScheduling
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class}, scanBasePackages = {
        "org.nebula", "com.opticalcable.graphqueryservice", "com.base.common"})
public class GraphQueryServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(GraphQueryServiceApplication.class, args);
    }

}
~~~~

3. 配置缓存相关参数
~~~~xml
cache:
  # 缓存过期时间；单位：分钟
  expired-time: 1
  # 缓存最大数目
  maximum_size: 2000
  # 缓存命中率，用来作为缓存淘汰的判断依据
  hit_rate_threshold: 0.7
  # 特定时间内多次访问不被纳入缓存；单位：秒
  low-rate-threshold: 2000
  # 超过特定时间后的处理不被纳入缓存，故大于 low-rate-threshold 且小于 high-rate-threshold 时间范围内的多次访问才被纳入缓存
  high-rate-threshold: 10000
  # 最大堆内存百分比，超过则不被纳入缓存
  max-heap-memory-percentage: 1
~~~~

4. 使用
~~~~java
@DynamicCache(key = "Array.prototype.join.call(key, \"|\")", lowRateThreshold = 3000L, highRateThreshold = 60000L,
        tagCache = "OpticalCircuit, LogicalOpticalCircuit, Fiber, OpticalCableSegment, LinkSegment, Joint, GeoLocation")
@LogRecord
@CustomResponseAdvice(message = "（路径规划，带数据校准）基于光路追溯经过的管道段及其A、Z端设备经纬度")
@RequestMapping(value = "/geo/queryPipeSectionsCoordinateByOpticalCircuitName", method = RequestMethod.GET)
public List<Waypoint> queryGeoPipeSectionsInfoByOpticalCircuitName(@RequestParam("opticalCircuitName") String opticalCircuitName) {
    return geoACLInterceptorService.queryGeoPipeSectionsInfoByOpticalCircuitName(opticalCircuitName);
}
~~~~

~~~~java
/*
 * Copyright (c) Huawei Technologies Co. Ltd. 2023-2023. All rights reserved.
 */

package com.base.common.aop;

import com.base.common.annotation.DynamicCache;
import com.base.common.util.rule.RuleProcessorUtil;
import com.base.common.util.trace.LogConstants;

import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;

import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import org.apache.commons.lang3.ObjectUtils;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;

import lombok.extern.slf4j.Slf4j;

import javax.annotation.PostConstruct;

import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

/**
 * description: 动态缓存切面
 *
 * @author l30041564
 * @since 2023-11-22
 */
@Slf4j
@Aspect
@Component
public class DynamicCacheAspect {

    /**
     * description: 缓存过期时间；单位：分钟
     */
    @Value("${cache.expired-time:10}")
    private long expiredTime;

    /**
     * description: 最大堆内存百分比
     */
    @Value("${cache.max-heap-memory-percentage:10}")
    private int maxHeapMemoryPercentage;

    /**
     * description: 最大缓存数目
     */
    @Value("${cache.maximum_size:2000}")
    private int maximumSize;

    /**
     * description: 命中率阈值
     */
    @Value("${cache.hit_rate_threshold:0.7}")
    private double hitRateThreshold;

    /**
     * description: 小于等于该时间内，不纳入缓存
     */
    @Value("${cache.low-rate-threshold:2000}")
    private long lowRateThreshold;

    /**
     * description: 大于该时间，不纳入缓存
     */
    @Value("${cache.high-rate-threshold:10000}")
    private long highRateThreshold;

    /**
     * description: 使用Guava的Cache作为缓存实现
     */
    public Cache<String, Object> cache;

    private final Map<String, Integer> counterMap = new ConcurrentHashMap<>();

    private final Map<String, Long> lastRequestMap = new ConcurrentHashMap<>();

    @PostConstruct
    public void initCache() {
        cache = CacheBuilder.newBuilder()
                .expireAfterWrite(expiredTime, TimeUnit.MINUTES)
                .maximumSize(maximumSize)
                .build();
    }

    @Around("@annotation(dynamicCache)")
    public Object handleDynamicCache(ProceedingJoinPoint joinPoint, DynamicCache dynamicCache) throws Throwable {
        String key = generateKey(joinPoint, dynamicCache);
        log.info(LogConstants.HIGH_LEVEL_SPECIFICATION_VARIABLE + "cache: {}", cache.asMap());
        Object result = cache.getIfPresent(key);
        if (ObjectUtils.allNotNull(result)) {
            return result;
        }
        if (!handleDuplicateRequest(key, dynamicCache)) {
            return joinPoint.proceed();
        }
        // 使用Guava Cache的get方法，如果缓存项不存在则执行方法并将结果缓存起来
        return cache.get(key, () -> {
            try {
                return joinPoint.proceed();
            } catch (Throwable e) {
                throw new RuntimeException(e);
            }
        });
    }

    /**
     * description: 生成缓存 key
     * @param joinPoint {@link org.aspectj.lang.JoinPoint}
     * @param dynamicCache {@link com.base.common.annotation.DynamicCache}
     * @return String 缓存key
     */
    private String generateKey(ProceedingJoinPoint joinPoint, DynamicCache dynamicCache) {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        String interfaceIdentifier = method.getDeclaringClass().getName() + "." + method.getName();

        Object[] args = joinPoint.getArgs();
        String[] argsStr = Arrays.stream(args)
                .map(Object::toString)
                .toArray(String[]::new);
        Object key = RuleProcessorUtil.processRule(dynamicCache.key(), argsStr, "key");
        return interfaceIdentifier + "_" + key.toString();
    }

    /**
     * description: 基于缓存命中率的驱逐
     * @param key 缓存key
     * @param dynamicCache {@link com.base.common.annotation.DynamicCache}
     * @return boolean 是否纳入缓存
     */
    public boolean handleDuplicateRequest(String key, DynamicCache dynamicCache) {
        // 获取当前时间戳
        long currentTime = System.currentTimeMillis();

        // 获取上一次请求的时间戳
        Long lastRequestTime = lastRequestMap.get(key);

        // 如果上一次请求时间戳存在且与当前时间戳的差值小于1秒，则计数器加1
        if (lastRequestTime != null) {
            lowRateThreshold = dynamicCache.lowRateThreshold() != 0L ? dynamicCache.lowRateThreshold() : lowRateThreshold;
            if (currentTime - lastRequestTime <= lowRateThreshold) {
                log.info(LogConstants.HIGH_LEVEL_SPECIFICATION_VARIABLE + "Dynamic Cache: {}", "Multiple accesses within 2s");
                return false;
            }
            highRateThreshold = dynamicCache.highRateThreshold() != 0L ? dynamicCache.highRateThreshold() : highRateThreshold;
            if (currentTime - lastRequestTime > highRateThreshold) {
                counterMap.remove(key);
                lastRequestMap.remove(key);
                log.info(LogConstants.HIGH_LEVEL_SPECIFICATION_VARIABLE + "Dynamic Cache: {}", "Visit after more than 5 seconds");
                return false;
            }
            // 滑窗计数
            int count = counterMap.merge(key, 1, Integer::sum);
            // 是否需要纳入缓存
            boolean isDuplicateRequest = count > 1;
            // 维护缓存标签
            if (isDuplicateRequest && StringUtils.isNotBlank(dynamicCache.tagCache())) {
                List<String> tagCache = Arrays.asList(dynamicCache.tagCache().split(",\\s*"));
                log.info(LogConstants.HIGH_LEVEL_SPECIFICATION_VARIABLE + "tag Cache: {}; key: {};", tagCache, key);
                cache.put(dynamicCache.tagCache(), key);
            }
            log.info(LogConstants.HIGH_LEVEL_SPECIFICATION_VARIABLE + "Dynamic Cache: {}", "Multiple accesses within 2s and 5s");
            // 如果计数超过阈值，将请求加入缓存
            return isDuplicateRequest;
        }

        // 更新上一次请求时间戳为当前时间戳
        lastRequestMap.put(key, currentTime);

        // 如果是第一次请求或者与上一次请求时间戳的差值大于等于1秒，则重置计数器为1
        counterMap.put(key, 1);
        log.info(LogConstants.HIGH_LEVEL_SPECIFICATION_VARIABLE + "Dynamic Cache: {}", "First visit");
        log.info(LogConstants.HIGH_LEVEL_SPECIFICATION_VARIABLE + "counterMap: {}, lastRequestMap: {}, cache: {};",
                counterMap, lastRequestMap, cache.asMap());
        // 异步进行内存淘汰
        CompletableFuture.runAsync(this::evictBasedOnHitRate);
        return false;
    }

    /**
     * description: 基于缓存命中率的驱逐
     *
     */
    private void evictBasedOnHitRate() {
        // 获取最大堆内存和当前堆内存的使用情况
        long maxHeapMemory = Runtime.getRuntime().maxMemory();
        long currentHeapMemory = Runtime.getRuntime().totalMemory();
        int currentHeapMemoryPercentage = (int) ((currentHeapMemory * 100) / maxHeapMemory);

        // 如果当前堆内存使用率超过阈值，则进行缓存淘汰
        if (currentHeapMemoryPercentage >= maxHeapMemoryPercentage) {
            // 计算需要清理的缓存项数量
            int cleanupSize = (int) (counterMap.size() * 0.1);

            // 获取即将过期的缓存项，并按照过期时间排序
            List<Map.Entry<String, Long>> entriesToCleanup = lastRequestMap.entrySet().stream()
                    .sorted(Map.Entry.comparingByValue())
                    .limit(cleanupSize)
                    .collect(Collectors.toList());

            // 遍历并清理缓存项
            entriesToCleanup.forEach(entry -> {
                String key = entry.getKey();
                counterMap.remove(key);
                lastRequestMap.remove(key);
            });
        }
        log.info(LogConstants.HIGH_LEVEL_SPECIFICATION_VARIABLE + "maxHeapMemory: {}, currentHeapMemory: {}, currentHeapMemoryPercentage: {};",
                maxHeapMemory, currentHeapMemory, currentHeapMemoryPercentage);
    }

}

~~~~

