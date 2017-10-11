---
layout:     post
title:      "[Spring Cloud] Eureka 的自我保护模式及相关问题"
subtitle:   "[Spring Cloud] Eureka 的自我保护模式及相关问题"
date:       2017-10-11 19:50:00
author:     "Kent"
header-img: "img/post-bg-2016.jpg"
catalog: true
tags:
    - eureka
---

[TOC]

## 一、Eureka 的自我保护模式

访问`Eureka`主页时，如果看到这样一段大红色的句子：

> <font color="red">**EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.**</font>

那么就表明着`Eureka`的 **自我保护模式(self-preservation mode)** 被启动了，当 Eureka Server 节点在短时间内丢失了过多实例的连接时（比如网络故障或频繁的启动关闭客户端），那么这个节点就会进入自我保护模式，一旦进入到该模式，Eureka server 就会保护服务注册表中的信息，不再删除服务注册表中的数据（即不会注销任何微服务），当网络故障恢复后，该 Ereaka Server 节点就会自动退出自我保护模式（我的 Eureka Server 已经几个月了，至今未自动退出该模式）

默认情况下，如果 Ereaka Server 在一段时间内没有接受到某个微服务示例的心跳，便会注销该实例（默认90秒），而一旦进入自我保护模式，那么即使你关闭了指定实例，仍然会发现该 Ereaka Server 的注册实例中会存在被关闭的实例信息，如果你对该实例做了负载均衡，那么仅关闭了其中一个实例，则通过网关调用接口`api`时很可能会发生如下异常：

```
{
    "timestamp": 1507707671780,
    "status": 500,
    "error": "Internal Server Error",
    "exception": "com.netflix.zuul.exception.ZuulException",
    "message": "GENERAL"
}
```

解决这种情况的方法主要有两种：

### 1. 关闭 Eureka 的自我保护模式

在`yml`配置文件中新增如下配置：

```yml
eureka:
  server:
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 4000 # This is not required
```

从根源解决问题，但是并不推荐在`PRD`环境中使用，后面会说明。

### 2. 重启 Eureka Server

通常而言，`PRD` 环境建议对 Eureka Server 做负载均衡，这样在依次关闭并开启 Eureka Server 后，无效的实例会被清除，并且不会对正常的使用照成影响。

## 二、开发环境的 Eureka Server

对于开发环境的 Eureka Server，个人更建议关闭它的自我保护模式，因为你可能需要不断的开启与关闭实例，如果并未关闭自我保护模式，那么很容易就会触发自我保护模式，此时对调试会相对比较麻烦。

但是关闭自我保护模式，会有另外一个可能的问题，即隔一段时间后，可能会发生实例并未关闭，却无法通过网关访问了，此时很可能是由于网络问题，导致实例（或网关）与 Eureka Server 断开了连接，Eureka Server 已经将其注销（网络恢复后，实例并不会再次注册），此时重启 Eureka Server 节点，并等待一小段时间即可。


## 三、参考链接

- [https://github.com/Netflix/eureka/wiki/Understanding-Eureka-Peer-to-Peer-Communication](https://github.com/Netflix/eureka/wiki/Understanding-Eureka-Peer-to-Peer-Communication)
- [http://www.itmuch.com/spring-cloud-sum/understanding-eureka-self-preservation/](http://www.itmuch.com/spring-cloud-sum/understanding-eureka-self-preservation/)
- [https://github.com/spring-cloud/spring-cloud-netflix/issues/2179](https://github.com/spring-cloud/spring-cloud-netflix/issues/2179)