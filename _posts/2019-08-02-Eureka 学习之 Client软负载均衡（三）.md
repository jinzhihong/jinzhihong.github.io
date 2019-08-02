---
layout:     post
title:      Eureka 学习之 Client软负载均衡（三）
subtitle:   Eureka 学习之 Client软负载均衡（三）
date:       2019-08-02
author:     金志宏
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java
    - Eureka
    - Spring Cloud
    - 微服务
---
# Eureka 学习之 Client软负载均衡（三）

这篇主要介绍 Eureka Client 是怎么实现软负载均衡的（纯demo演示非源码层面），我的个人习惯，先跑一整编Demo 最后再研究源码，源码放到最后边学边记录。

## 1. Eureka Client 多节点

这里就不记录怎么创建 Server 了，有兴趣的同学可以看我的前2篇 Eureka 学习文章。一般的微服务架构会根据业务进行服务拆分，单服务器多节点的情况下大多数的部署方法如下表

| 业务模块 | 应用程序名   | 端口号      |
| -------- | ------------ | ----------- |
| 用户管理 | service-user | 20001-20010 |
| 部门管理 | service-dept | 20011-20020 |
| …        | …            | …           |

下面就实战操作一下

### 1.1 创建 Eureka Client  service-user

我使用了`Idea Spring Initializr `创建的项目，就不做截图了。 File-New-project 选择 Spring Initializr 插件。

- 添加`Eureka Discovery Client` 依赖
- 添加`Spring Web starter`依赖。
- 添加`OpenFeign` 依赖

1. Maven 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.blackhold</groupId>
    <artifactId>spring-cloud-eureka-consumer</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-cloud-eureka-consumer</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR2</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```

2. 配置 Properties 我们不需要配 server.port 因为不同节点的

```properties
# Eureka 客户端口号
# server.port=20001
# 应用名
spring.application.name=provider-user
# Eureka 相关配置
# 1.注册地址 注册到 Eureka Server 服务
eureka.client.service-url.defaultZone=http://localhost:10001/eureka/
```

3. 编写接口Controller为了方便查看软负载结果，加入了启动环境变量中port的打印

```java
@RestController
public class UserController {

    @Autowired
    Environment environment;

    @GetMapping("/getUser/{id}")
    public String getUser(@PathVariable() String id){
        return "port: " + environment.getProperty("server.port") + " | This is User [id = " + id + "]";
    }
}
```

4. 配置Idea启动的`Spring boot`的环境变量，分别设置启动port为20001，20002，然后启动应用

![e0iP1A.png](https://s2.ax1x.com/2019/08/02/e0iP1A.png)

5. 启动类添加注解，如果需要用`Feign`功能需要添加 `@EnableFeignClients`注解

```java
@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudEurekaProviderApplication {

    public static void main(String[] args) {

        SpringApplication.run(SpringCloudEurekaProviderApplication.class, args);
    }

}
```

6. 启动后查看注册中心，可以看见一个实例注册了2个服务分别在20001和20002端口

[![e0iNh4.png](https://s2.ax1x.com/2019/08/02/e0iNh4.png)](https://imgchr.com/i/e0iNh4)

测试接口 http://localhost:20001/getUser/2  返回 port: 20001 | This is User [id = 2]

测试接口 http://localhost:20002/getUser/1  返回 port: 20002 | This is User [id = 1]



### 1.2 创建 Eureka Client service-dept

创建项目方法和上面相同，端口号定义了 20011，20012，直接看代码

1. Maven 配置，同上
2. properties 配置，同上

```properties
# Eureka 客户端口号
# server.port=20011
# 应用名
spring.application.name=service-dept
# Eureka 相关配置
# 1.注册地址 注册到 Eureka Server 服务
eureka.client.service-url.defaultZone=http://localhost:10001/eureka/
```

3. 添加接口

```java
// 注意这里的value 等同于 服务提供者的 的application.name
@FeignClient(value = "service-user")
public interface UserService {
    @RequestMapping(value = "/getUser/{id}", method = RequestMethod.GET)
    String getUser(@PathVariable("id") String id);
}
```

4. 添加 Controller 调用接口 

```java
@RestController
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/getUser/{id}")
    public String getUser(@PathVariable String id) {
				//测试软负载调用100次
        for (int i = 0; i < 100; i++) {
            System.out.println(userService.getUser(id));
        }
        return "ok";
    }
}
```

5. run 方法启动项目

```java
@SpringBootApplication
@EnableFeignClients  //如果使用FeignClient调用微服务需要添加此注解，并且会自动注册到Eureka
public class SpringCloudEurekaConsumerApplication  {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudEurekaConsumerApplication.class, args);
    }

}
```

6. 启动成功后，可以看到注册中心，注册了 service-user 服务和 service-dept 服务。

![e0E9VP.png](https://s2.ax1x.com/2019/08/02/e0E9VP.png)

7. 测试 调用http://localhost:20011/getUser/2 或者 http://localhost:20012/getUser/2则会输出

port: 20001 | This is User [id = 2]
port: 20002 | This is User [id = 2]
port: 20001 | This is User [id = 2]
port: 20002 | This is User [id = 2]
port: 20001 | This is User [id = 2]

…省略95行

输出结果很平均，基本情况下 1：1的调用 也就是一个请求走的是20001一个请求走但是20002：

## 3. 总结

通过application-name名称可以调用服务集群，调用过程中 Eureka 会自动实现软负载。

## 3. 参考

> [Spring Cloud netflix 英文官网][https://spring.io/projects/spring-cloud-netflix]

