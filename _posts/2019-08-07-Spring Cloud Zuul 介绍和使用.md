---
layout:     post
title:      Spring Cloud Zuul 介绍和使用
subtitle:   Spring Cloud Zuul 介绍和使用
date:       2019-08-07
author:     金志宏
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java
    - Netflix
    - Spring Cloud
    - 微服务
    - Zuul
---
#Spring Cloud Zuul 介绍和使用

## 1. 前言：路由器和过滤器 Zuul

​		路由是微服务架构不可或缺的一部分。例如，`/`可以映射到您的Web应用程序，`/api/users`映射到用户服务，`/api/shop`映射到商店服务。`Zuul`的`Netflix`基于JVM的路由器和服务器负载均衡器。

​		微服务架构我们有很多服务，每个服务拥有不同的IP地址，端口，服务名称。这些服务的调用路径没法统一管理，前端调用服务需要知道每个服务的地址。`Zuul`路由的作用就在这里。`Zuul`可以提供一个统一调用地址，通过映射配置，发现服务。前端服务调用者再也不需要关心地址或端口了。由 `Zuul` 统一代理。

## 2. Quick Start

​		新建一个`SpringBoot`项目，`Maven`配置，`Application` 中添加 `@EnableZuulProxy` 注解。这样就可以以最低配置启动`Zull`

```xml
				<!-- zuul-->
				<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
```

```java
@SpringBootApplication
@EnableZuulProxy
public class GetewayZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(GetewayZuulApplication.class, args);
    }

}
```

```properties
# 路由端口
server.port=8088
#给要路由的API请求加前缀 可加可不加
zuul.prefix=/v1
# user服务  zuul.routes.XX.path  zuul.routes.XX.url  XX自定义，P是路由请求，映射到物理地址
# 配置了zuul之后,那么整个分布式系统，对外则只暴露http://zuul-IP:zuul-port/v1/服务自定义地址/具体
# API请求地址
# 用户服务
zuul.routes.user.path=/user-service/**
zuul.routes.user.url=http://localhost:20001/
# 部门服务
zuul.routes.dept.path=/dept-service/**
zuul.routes.dept.url=http://localhost:20011/
```

我的`Eureka`注册中心如图，`192.168.0.8`是我本地IP，user-service 服务2个节点，端口分别是20001和20002，dept-service 一个节点，端口20011

[![msmrsH.png](https://s2.ax1x.com/2019/08/23/msmrsH.png)](https://imgchr.com/i/msmrsH)

user-service 中暴露一个接口

```java
@RestController
public class UserController {

    @Autowired
    Environment environment;

    @GetMapping("/getUser/{id}")
    public String getUser(@PathVariable() String id) {
        return "port: " + environment.getProperty("server.port") + " | This is User [id = " + id + "]";
    }
}
```

启动`Zuul`后，通过URL `http://localhost:8088/v1/user-service/getUser/4` ，即可获得正确输出，因为代理了20001端口，所以输出如下：

```properties
# 如果配置 zuul.routes.user.url=http://localhost:20001/
port: 20001 | This is User [id = 4]
# 如果配置 zuul.routes.user.url=http://localhost:20002/
port: 20002 | This is User [id = 4]
```

这种配置有个缺陷，就是只能代理物理地址，如果是单服务多节点，或者换了IP地址，那么就要重新配置。`Zuul`还有一种配置可以针对服务名`application`进行配置。代理具体服务，通过`serviceId`即可，但是需要让`Zuul`注册到`Eureka Server`中才可以发现服务。`Zuul`是没有注册功能的，所以我们需要添加 `Eureka Client`依赖。

```xml
				<!--eureka-client-->
				<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

```java
@SpringBootApplication
@EnableZuulProxy
@EnableDiscoveryClient //添加注解
public class GetewayZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(GetewayZuulApplication.class, args);
    }

}
```

```properties
# 路由端口
server.port=8088
# 注册服务名
spring.application.name=ms-gateway
# 注册中心地址
eureka.client.service-url.defaultZone=http://node1:10001/eureka/
#网关最大超时时间10s 默认2000毫秒
zuul.host.connect-timeout-millis=10000
#网关最大连接数 默认200
zuul.host.max-total-connections=200
#给要路由的API请求加前缀 可加可不加
zuul.prefix=/v1
# user服务  zuul.routes.XX.path  zuul.routes.XX.url  XX自定义，P是路由请求，映射到物理地址
# 配置了zuul之后,那么整个分布式系统，对外则只暴露http://zuul-IP:zuul-port/v1/服务自定义地址/具体
# API请求地址
# 用户服务
zuul.routes.user.path=/user-service/**
# 发现用户服务并代理
zuul.routes.user.serviceId=user-service
# 部门服务
zuul.routes.dept.path=/dept-service/**
# 发现部门服务并代理
zuul.routes.dept.serviceId=dept-service
```

启动`Spring Boot`，通过URL `http://localhost:8088/v1/user-service/getUser/4` 

```properties
# 如果配置 zuul.routes.user.serviceId=user-service 则是代理整个User服务
# 第一次请求
port: 20001 | This is User [id = 4]
# 第二次请求
port: 20002 | This is User [id = 4]
# 第三次请求
port: 20001 | This is User [id = 4]
```

这样就实现了路由通过服务名称发现服务，并动态代理。



>[Spring Cloud Netflix Zuul](https://cloud.spring.io/spring-cloud-netflix/reference/html/#_router_and_filter_zuul)