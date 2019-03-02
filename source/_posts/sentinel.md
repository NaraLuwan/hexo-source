---
title: sentinel 工作原理
tags: [sentinel]
categories:
  - Java
  - web
date: 2019-03-01 19:43:50
description: Source Code Analysis Series
---
<p class="description"></p>

## Sentinel 简介

Sentinel是阿里中间件团队开源的，面向分布式服务架构的**轻量级高可用**流量控制组件，主要以**流量**为切入点，从流量控制、熔断降级、系统负载保护等多个维度来帮助用户保护服务的稳定性。

## Quick Start

### 接入步骤

1. 引入 Sentinel 依赖
```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-core</artifactId>
    <version>1.4.2</version>
</dependency>
```
2. 定义资源

可以是程序中的任何内容。如：调用其他应用提供的服务，或一个方法、一段代码

3. 设置规则

针对资源设置限流策略和对应指标的阈值。如：最大QPS、最大线程数

<!-- more -->

### 实例

![定义资源](/uploads/sentinel/sentinel01.png)

![设置规则](/uploads/sentinel/sentinel02.png)

SphU.entry()方法会去申请一个entry，如果能够申请成功，则说明没有被限流，否则会抛出BlockException，表明已经达到阈值被限流了。

## 工作原理

在 Sentinel 里面，所有的资源都对应一个资源名称以及一个 Entry。Entry 可以通过对主流框架的适配自动创建，也可以通过注解的方式或调用 API 显式创建；每一个 Entry 创建的时候，同时也会创建一系列功能插槽（slot chain）。
默认有以下插槽（用户也可以自定义）：

### 责任链模式
NodeSelectorSlot 负责收集资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；

ClusterBuilderSlot 负责存储资源的统计信息以及调用者信息。如 RT, QPS, thread count 等，这些信息将用作为多维度限流，降级的依据；

StatisticSlot 负责记录、统计不同纬度的 runtime 指标监控信息；

FlowSlot 负责根据预设的限流规则以及前面 slot 统计的状态，来进行流量控制；

AuthoritySlot 负责根据配置的黑白名单和调用来源信息，来做黑白名单控制；

DegradeSlot 负责通过统计信息以及预设的规则，来做熔断降级；

SystemSlot 负责通过系统的状态，例如 load1 等，来控制总的入口流量；

### 总体框架

![总体框架](/uploads/sentinel/sentinel03.png)

## 源码解析

创建资源的时候会去初始化一个对应的上下文，上下文会保存责任链的跟节点，之后每次调用 entry 方法时会依次调用每个节点的 entry 方法，同一个资源可以创建多条规则，Sentinel会对该资源的所有规则依次遍历，直到有规则触发限流抛出BlockException或者所有规则遍历完毕返回entry。

关键源码：

```java
public class CtSph implements Sph {
    private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
            throws BlockException {
            // 获取该资源对应的上下文
            Context context = ContextUtil.getContext();
            if (context instanceof NullContext) {
                // here init the entry only. No rule checking will be done.
                return new CtEntry(resourceWrapper, null, context);
            }
    
            if (context == null) {
                context = MyContextUtil.myEnter(Constants.CONTEXT_DEFAULT_NAME, "", resourceWrapper.getType());
            }
    
            // Global switch is close, no rule checking will do.
            if (!Constants.ON) {
                return new CtEntry(resourceWrapper, null, context);
            }
    
            // 获取责任链（如果没有就新建）
            ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);
    
            // 这里为空只有一种可能：责任链缓存(map)超过长度限制。默认 MAX_SLOT_CHAIN_SIZE = 6000
            if (chain == null) {
                return new CtEntry(resourceWrapper, null, context);
            }
    
            // 初始化一个 entry
            Entry e = new CtEntry(resourceWrapper, chain, context);
            try {
                // 开始依次调用责任链的 entry 方法
                chain.entry(context, resourceWrapper, null, count, prioritized, args);
            } catch (BlockException e1) {
                // 如果 exit 的时候发生了异常呢？
                e.exit(count, args);
                throw e1;
            } catch (Throwable e1) {
                // This should not happen, unless there are errors existing in Sentinel internal.
                RecordLog.info("Sentinel unexpected exception", e1);
            }
            return e;
        }
}
```

接下来看下创建责任链的流程：

```java
public class CtSph implements Sph {
    ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
            // 根据资源名缓存责任链
            ProcessorSlotChain chain = chainMap.get(resourceWrapper);
            if (chain == null) {
                synchronized (LOCK) {
                    chain = chainMap.get(resourceWrapper);
                    // double check
                    if (chain == null) {
                        if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                            return null;
                        }
                        // 默认的初始化方法
                        chain = SlotChainProvider.newSlotChain();
                        Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(chainMap.size() + 1);
                        newMap.putAll(chainMap);
                        newMap.put(resourceWrapper, chain);
                        chainMap = newMap;
                    }
                }
            }
            return chain;
        }
}
```

```java
public class DefaultSlotChainBuilder implements SlotChainBuilder {

    @Override
    public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();
        chain.addLast(new NodeSelectorSlot());
        chain.addLast(new ClusterBuilderSlot());
        chain.addLast(new LogSlot());
        chain.addLast(new StatisticSlot());
        chain.addLast(new SystemSlot());
        chain.addLast(new AuthoritySlot());
        chain.addLast(new FlowSlot());
        chain.addLast(new DegradeSlot());

        return chain;
    }

}
```

接下来看是怎么实现依次调用责任链的 entry 方法的：

```java
// 其实是一个链表
public class DefaultProcessorSlotChain extends ProcessorSlotChain {

    // 重写第一个节点
    AbstractLinkedProcessorSlot<?> first = new AbstractLinkedProcessorSlot<Object>() {

        @Override
        public void entry(Context context, ResourceWrapper resourceWrapper, Object t, int count, boolean prioritized, Object... args)
            throws Throwable {
            // 注意这里直接调用父类的 fireEntry 方法
            super.fireEntry(context, resourceWrapper, t, count, prioritized, args);
        }

        @Override
        public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
            super.fireExit(context, resourceWrapper, count, args);
        }

    };
    AbstractLinkedProcessorSlot<?> end = first;

    @Override
    public void addFirst(AbstractLinkedProcessorSlot<?> protocolProcessor) {
        protocolProcessor.setNext(first.getNext());
        first.setNext(protocolProcessor);
        if (end == first) {
            end = protocolProcessor;
        }
    }

    @Override
    public void addLast(AbstractLinkedProcessorSlot<?> protocolProcessor) {
        end.setNext(protocolProcessor);
        end = protocolProcessor;
    }
    
    @Override
        public void entry(Context context, ResourceWrapper resourceWrapper, Object t, int count, boolean prioritized, Object... args)
            throws Throwable {
        first.transformEntry(context, resourceWrapper, t, count, prioritized, args);
    }
}
```

那再继续看父类的 fireEntry 方法：

```java
public abstract class AbstractLinkedProcessorSlot<T> implements ProcessorSlot<T> {

    private AbstractLinkedProcessorSlot<?> next = null;

    @Override
    public void fireEntry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args)
        throws Throwable {
        // 执行 next 节点的 transformEntry 方法
        if (next != null) {
            next.transformEntry(context, resourceWrapper, obj, count, prioritized, args);
        }
    }

    @SuppressWarnings("unchecked")
    void transformEntry(Context context, ResourceWrapper resourceWrapper, Object o, int count, boolean prioritized, Object... args)
        throws Throwable {
        T t = (T)o;
        // 每个 slot 节点自己实现
        entry(context, resourceWrapper, t, count, prioritized, args);
    }
}
```

到这里就很清楚了，生成的 chain 实际是一个链表，调用 entry 方法的时候先从链表的 first 节点开始，每个节点先调用自己的 entry 方法做完自己的事情后调用 next 节点的 entry 
方法，直到有节点触发限流抛出BlockException或者所有节点处理完正常返回 entry。

<hr />