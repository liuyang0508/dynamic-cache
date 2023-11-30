# dynamic-cache
动态缓存

## DynamicCacheAspect切面

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

