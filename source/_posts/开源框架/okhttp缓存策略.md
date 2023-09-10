---
title: okhttp缓存策略介绍
date: 2023-09-10 15:35:14
tags:
categories:
---

<meta name="referrer" content="no-referrer" />

在 okhttp 责任链中在建立网络连接之前有个 CacheInterceptor 负责缓存相关。
okhttp 实现了 http 协议的所有缓存策略和缓存字段的解析。
缓存策略实现逻辑在 `CacheStrategy` 类中。
缓存信息实现逻辑和容器在 `Cache` 类中。


