---
title: guavacache 缓存策略选择
tags: [guava]
categories:
  - Java
date: 2017-03-07 12:36:00
---

### 三个参数

- expireAfterAccess: 当缓存项在指定的时间段内没有被读或写就会被回收。
- expireAfterWrite：当缓存项在指定的时间段内没有更新就会被回收。
- refreshAfterWrite：当缓存项上一次更新操作之后的多久会被刷新。

<!-- more -->  

expireAfterWrite 使每次更新之后的指定时间让缓存失效，然后重新加载缓存。guava cache 会严格限制只有1个加载操作，这样会很好地防止缓存失效的瞬间大量请求穿透到后端引起雪崩效应。

通过分析源码，guava cache 在限制只有1个加载操作时进行加锁，其他请求必须阻塞等待这个加载操作完成；而且，在加载完成之后，其他请求的线程会逐一获得锁，去判断是否已被加载完成，每个线程必须轮流地走一个“”获得锁，获得值，释放锁“”的过程，这样性能会有一些损耗。

refreshAfterWrite 在refresh的过程中，严格限制只有1个重新加载操作，而其他查询先返回旧值，这样有效地可以减少等待和锁争用。

但它也有一个缺点，因为到达指定时间后，它不能严格保证所有的查询都获取到新值。 guava cache 并没使用额外的线程去做定时清理和加载的功能，而是依赖于查询请求。在查询的时候去比对上次更新的时间，如超过指定时间则进行加载或刷新。

所以，如果使用refreshAfterWrite，在吞吐量很低的情况下，如很长一段时间内没有查询之后，发生的查询有可能会得到一个旧值（这个旧值可能来自于很长时间之前）。

可以看出 refreshAfterWrite 和 expireAfterWrite 两种方式各有优缺点，各有使用场景。那么能否在 refreshAfterWrite 和 expireAfterWrite 找到一个折中？

比如说控制缓存每1s进行 refresh，如果超过2s没有访问，那么则让缓存失效，下次访问时不会得到旧值，而是必须得待新值加载。

google 到这种方案亲测有效：

```java
CacheBuilder.newBuilder().maximumSize(1000).refreshAfterWrite(1, TimeUnit.SECONDS).expireAfterWrite(2, TimeUnit.SECONDS).build();
```

