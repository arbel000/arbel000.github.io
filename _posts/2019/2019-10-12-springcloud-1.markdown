---
layout:     post
title:      "Spring Cloud框架(一) 概况"
date:       2019-10-12
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - springCloud
--- 

[spring Cloud框架(一) 概况](https://zhouj000.github.io/2019/10/12/springcloud-1/)  



> 1.第一代服务框架  
> 代表：Dubbo(Java)、Orleans(.Net)等  
> 特点：和语言绑定紧密  
> 2.第二代服务框架  
> 代表：Spring Cloud等  
> 现状：适合混合式开发(例如借助Steeltoe OSS可以让ASP.Net Core与Spring Cloud集成)  
> 3.第三代服务框架  
> 代表：Service Mesh(服务网格) => 例如Service Fabric、lstio、Linkerd、Conduit等  
> 现状：在快速发展中，更新迭代比较快

未来可能的主流服务架构和技术栈(**云原生(Cloud Native)**及**Service Mesh**时代)：
![layer](/img/in-post/2019/10/layer.jpg)
基础的**云平台**为微服务提供了资源能力(计算、存储和网络等)，**容器**作为最小工作单元被Kubernetes调度和编排，**Service Mesh**(服务网格)管理微服务的服务通信，最后通过**API Gateway**向外暴露微服务的业务接口

[一个可供中小团队参考的微服务架构技术栈](https://www.infoq.cn/article/china-microservice-technique/?utm_source=tuicool&utm_medium=referral)

从这篇文章可以看到，Spring Cloud技术栈中的有些组件离生产级开发尚有一定距离，所以现在大多数公司会选择**部分拥抱**Spring Cloud


# Spring Cloud核心子项目

我觉得Spring Cloud最大的优点就是组件丰富，功能齐全

+ Spring Cloud Netflix：核心组件，可以对多个Netflix OSS开源套件进行整合
	- **Zuul**：网关组件，提供智能路由、访问过滤等功能
	- **Eureka**：服务治理组件，包含服务注册与发现
		+ 由于Eureka官宣2.x版本不再开源，因此对于大厂肯定是自研服务注册中心的；而中小公司会使用其他的开源注册中心，比如Consul、ZooKeeper、Etcd、CoreDNS、Nacos(+dubbo，Spring Cloud Eureka + Spring Cloud Config<config server>)等中间件
		+ 在微服务架构中，服务注册的方式其实大体上也只有两种，一种是使用Zookeeper和etcd等配置管理中心，另一种是使用DNS服务，比如说Kubernetes中的CoreDNS服务
	- **Ribbon**：客户端负载均衡的服务调用组件
	- **Hystrix**：容错管理组件，实现了熔断器
	- **Feign**：基于Ribbon和Hystrix的声明式服务调用组件
	- **Archaius**：外部化配置组件
+ Spring Cloud Config：配置管理工具，实现应用配置的外部化存储
	- 没有经过生产验证，其他竞品有Apollo、disconf、Nacos
+ Spring Cloud Bus：事件、消息总线，用于传播集群中的状态变化或事件，以及触发后续的处理
+ Spring Cloud Security：基于spring security的安全工具包
+ Spring Cloud Consul: 封装了Consul操作，与Docker容器可以无缝集成
+ Spring Cloud Sleuth: 微服务跟踪，与Zipkin的配合使用

首先，尽管Spring Cloud带有"Cloud"这个单词，但它**并不是云计算解决方案**，而是在**Spring Boot基础之上**构建的，用于快速构建分布式系统的通用模式的**工具集**

其次，使用Spring Cloud开发的应用程序非常适合在**Docker和PaaS**(比如Pivotal Cloud Foundry)上部署，所以又叫做**云原生应用**(Cloud Native Application，可以简单地理解为面向云环境的软件架构)

所以Spring Cloud是一个基于Spring Boot实现的**云原生应用开发工具**，它为基于JVM的云原生应用开发中涉及的配置管理、服务发现、熔断器、智能路由、微代理、控制总线、分布式会话和集群状态管理等操作提供了一种简单的开发方式



官方：[Spring Cloud](https://spring.io/projects/spring-cloud)  
中文网：[Spring Cloud](https://www.springcloud.cc/)  

