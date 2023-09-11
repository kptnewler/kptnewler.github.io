---
title: okhttp缓存策略介绍
date: 2023-09-10 15:35:14
tags:
categories:
---

<meta name="referrer" content="no-referrer" />

在 okhttp 责任链中在建立网络连接之前有个 CacheInterceptor 负责缓存相关。
okhttp 实现了 http 协议的所有缓存策略和缓存字段的解析。

在请求之前先调用 `CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()` 获取缓存策略。
```kotlin
    val call = chain.call()
    val cacheCandidate = cache?.get(chain.request())

    val now = System.currentTimeMillis()

    val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
    val networkRequest = strategy.networkRequest
    val cacheResponse = strategy.cacheResponse

    cache?.trackResponse(strategy)
    val listener = (call as? RealCall)?.eventListener ?: EventListener.NONE

    if (cacheCandidate != null && cacheResponse == null) {
      // The cache candidate wasn't applicable. Close it.
      cacheCandidate.body.closeQuietly()
    }
```

缓存策略实现逻辑在 `CacheStrategy` 类中。
缓存信息实现逻辑和容器在 `Cache` 类中。

## 初始化
在初始化的时候，如果有缓存响应，则从响应头中依次取出:
- Date, 服务器的时间。
- Expires，过期时间。
- Last-Modified: 资源被修改的日期，用于条件请求。
- ETag: 客户端资源唯一标识，主要是用来解决修改时间无法准确区分文件变化的问题，用于条件请求。
- Age: 资源在缓存中存在的时间。
```kotlin
// CacheStrategy.Factory()
    init {
      if (cacheResponse != null) {
        this.sentRequestMillis = cacheResponse.sentRequestAtMillis
        this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis
        val headers = cacheResponse.headers
        for (i in 0 until headers.size) {
          val fieldName = headers.name(i)
          val value = headers.value(i)
          when {
            fieldName.equals("Date", ignoreCase = true) -> {
              servedDate = value.toHttpDateOrNull()
              servedDateString = value
            }
            fieldName.equals("Expires", ignoreCase = true) -> {
              expires = value.toHttpDateOrNull()
            }
            fieldName.equals("Last-Modified", ignoreCase = true) -> {
              lastModified = value.toHttpDateOrNull()
              lastModifiedString = value
            }
            fieldName.equals("ETag", ignoreCase = true) -> {
              etag = value
            }
            fieldName.equals("Age", ignoreCase = true) -> {
              ageSeconds = value.toNonNegativeInt(-1)
            }
          }
        }
      }
    }
```

`computeCandidate()` 是计算缓存策略的核心逻辑

`Cache-Controller: only-if-cached` 只能用缓存，条件请求不为空，则直接返回 null。 

```kotlin
fun compute(): CacheStrategy {
    val candidate: CacheStrategy = computeCandidate()

    // We're forbidden from using the network and the cache is insufficient.
    if (candidate.networkRequest != null && request.cacheControl.onlyIfCached) {
        return CacheStrategy(null, null)
    }

    return candidate
}
```

## CacheStrategy 处理缓存
主要看 `compute()` 里面处理逻辑，分为好几块。
### 是否可以缓存
1. 判断 response code 是否符合条件
2. 请求头是否有 Cache-Controller:max-age、private、public 和 Expires:value，没有直接当不需要缓存。
```kotlin
if (!isCacheable(cacheResponse, request)) {
    return CacheStrategy(request, null)
}
```

### 判断是否有条件请求
1. 如果缓存的 request 是条件请求，那么下次
2. Cache-Control:no-cache，表示没有任何缓存。
```kotlin
val requestCaching = request.cacheControl
if (requestCaching.noCache || hasConditions(request)) {
    return CacheStrategy(request, null)
}
```

如果没有条件请求头，但是缓存的响应头有条件请求的参数，根据请求参数生成条件请求头。
```kotlin
      val conditionName: String
      val conditionValue: String?
      when {
        etag != null -> {
          conditionName = "If-None-Match"
          conditionValue = etag
        }

        lastModified != null -> {
          conditionName = "If-Modified-Since"
          conditionValue = lastModifiedString
        }

        servedDate != null -> {
          conditionName = "If-Modified-Since"
          conditionValue = servedDateString
        }

        else -> return CacheStrategy(request, null) // No condition! Make a regular request.
      }
```

### 根据资源保质期和缓存时间

#### 资源缓存时间
ageMillis 表示资源已经缓存的时间，分成 3 部分，3 者相加即为资源缓存时间，因为资源可能在缓存服务器上或者 age=null && data = null，导致 receivedAge 为空。
虽然当 receivedAge 不为 0 时，前两点可能时间重合，但是为了防止用了过期的资源，这样计算最保险。
- receivedAge，资源运输过程的损耗时间。
- responseDuration，客户端请求响应时间，一个 RTT。
- residentDuration，资源本地缓存时间。

#### 资源保质期
freshMillis 保质期时间，服务端可能提供多个保质期，根据优先级，保质期计算公式主要是前两点。
- Cache-Controller 的 mag-age 优先级最高，返回值就是保质期。
- Expires 过期时间，当前时间 - Expires，计算得出保质期。
- Last-Modified， 如果资源的最后修改时间确定且没有查询参数，资源到达客户端时间 - 最后修改时间 的 10% 作为保质期。

#### minFreshMillis
Cache-Controller 的 min-fresh，最小新鲜时间，即保质期时间提前了多久了。

#### maxStaleMillis
Cache-Controller 的 max-stale 最大过期时间，即保质期时间延长多久。

最终计算方法 缓存时间 < 保质期 即可以使用缓存
ageMillis < freshMillis + maxStaleMillis - minFreshMillis
 
```kotlin
      val responseCaching = cacheResponse.cacheControl

      val ageMillis = cacheResponseAge()
      var freshMillis = computeFreshnessLifetime()

      if (requestCaching.maxAgeSeconds != -1) {
        freshMillis = minOf(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds.toLong()))
      }

      var minFreshMillis: Long = 0
      if (requestCaching.minFreshSeconds != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds.toLong())
      }

      var maxStaleMillis: Long = 0
      if (!responseCaching.mustRevalidate && requestCaching.maxStaleSeconds != -1) {
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds.toLong())
      }

      if (!responseCaching.noCache && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        val builder = cacheResponse.newBuilder()
        if (ageMillis + minFreshMillis >= freshMillis) {
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"")
        }
        val oneDayMillis = 24 * 60 * 60 * 1000L
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"")
        }
        return CacheStrategy(null, builder.build())
      }
```

## 响应缓存

如果条件请求，服务端返回 304 表示资源为修改，则可以直接使用缓存响应。

```kotlin
    if (cacheResponse != null) {
      if (networkResponse?.code == HTTP_NOT_MODIFIED) {
        val response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers, networkResponse.headers))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
            .cacheResponse(cacheResponse.stripBody())
            .networkResponse(networkResponse.stripBody())
            .build()

        networkResponse.body.close()

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache!!.trackConditionalCacheHit()
        cache.update(cacheResponse, response)
        return response.also {
          listener.cacheHit(call, it)
        }
      } else {
        cacheResponse.body.closeQuietly()
      }
    }
```

响应满足缓存条件，则添加到缓存中。
```kotlin
    if (cache != null) {
      if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        val cacheRequest = cache.put(response)
        return cacheWritingResponse(cacheRequest, response).also {
          if (cacheResponse != null) {
            // This will log a conditional cache miss only.
            listener.cacheMiss(call)
          }
        }
      }

      if (HttpMethod.invalidatesCache(networkRequest.method)) {
        try {
          cache.remove(networkRequest)
        } catch (_: IOException) {
          // The cache cannot be written.
        }
      }
    }

```