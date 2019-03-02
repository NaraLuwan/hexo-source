---
title: Apollo 架构分析
tags: [apollo]
categories:
  - Java
  - Web
date: 2018-12-26 22:00:48
description: Source Code Analysis Series
---
<p class="description"></p>

## 基础模型

![基础推拉模型](/uploads/apollo/basicArchitecture.png)

1. 用户在配置中心对配置进行修改并发布
2. 配置中心通知 Apollo 客户端有配置更新
3. Apollo 客户端从配置中心拉取最新的配置、更新本地配置并通知到应用

## 模块分析

下图是 Apollo 的七个模块，其中四个模块是和功能相关的核心模块，另外三个模块是辅助服务发现的模块：<!-- more -->

![架构设计](/uploads/apollo/architecture.png)

### Client
- Apollo 客户端，为应用获取配置，实时更新
- 通过 Meta Server 获取 Config Service 服务列表（IP+Port），通过 IP+Port 访问服务
- 在 Client 侧做 load balance、错误重试

### Portal
- 提供配置的管理后台界面
- 通过 Meta Server 获取 Admin Service 服务列表（IP+Port），通过 IP+Port 访问服务
- 在 Portal 侧做load balance、错误重试

### Config Service
- 给客户端（client）提供配置获取和更新推送接口（基于 Http long polling）

### admin service
- 给管理后台（portal） 提供配置管理接口（Http Restful API）

### Meta Server
- 封装服务发现的细节，只是一个逻辑角色

### Eureka
- 基于 Eureka 和 Spring Cloud Netflix 提供服务注册和发现, Config Service 和 Admin Service 会向 Eureka 注册服务，并保持心跳
- 为了简单起见，目前 Eureka 在部署时和 Config Service 是在一个 JVM 进程中的（通过 Spring Cloud Netflix）

### SLB
- 和域名系统配合，负载均衡

## 思考
> 封装 meta Server 的好处？
- 对 Portal 和 Client 而言，永远通过一个Http接口获取 Admin Service 和 Config Service 的服务信息
- 由于原生 Eureka 只支持 Java 客户端，封装 meta Server 可以将 Eureka 的服务发现接口以更简单明确的HTTP接口的形式暴露出来，方便其他语言接入

> 为什么选用 Eureka?
- 它提供了完整的 Service Registry 和 Service Discovery 实现，也经受住了 Netflix 自己的生产环境考验，相对使用起来会比较省心
- 和 Spring Cloud 无缝集成，为了提高配置中心的可用性和降低部署复杂度，需要尽可能地减少外部依赖
- 由于代码是开源的，非常便于我们了解它的实现原理和排查问题

## 参考资料

- [Apollo 官方文档](https://github.com/ctripcorp/apollo/wiki)
- [Apollo 开发指南](https://github.com/ctripcorp/apollo/wiki/Apollo%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97)
- [芋道源码](http://www.iocoder.cn/categories/Apollo/)
  
<hr />