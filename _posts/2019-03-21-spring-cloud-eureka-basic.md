---
layout: post
author: czb1n
title:  "学习SpringCloud之服务注册与发现Eureka"
date:   2019-03-21 14:31:23
categories: [Server]
tags: [SpringCloud, Server, Kotlin]
---

## 简介
- **Eureka**是**SpringCloudNetflix**微服务套件中的一个组件。  

负责服务的注册和发现。其中包含**EurekaServer**为服务端，即服务注册中心。以及**EurekaClient**，即各个注册的微服务。  
Eureka支持高可用的配置，支持自动保护模式，允许故障期间继续提供服务。

> 以下示例均基于SpringCloud的**Greenwich.SR1**版本。

## 基础依赖
创建项目后，先引入SpringCloud的主要依赖。
``` Html
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-function-context</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-task</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

再在需要Eureka的模块中引入EurekaServer的依赖。
``` Html
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
            <version>2.1.1.RELEASE</version>
        </dependency>
    </dependencies>
```

## EurekaServer

需启动一个SpringBoot的应用作为注册中心。  
在启动时通过**@EnableEurekaServer**来声明这个应用会作为基于Eureka的注册中心。  
``` Java
@SpringBootApplication
@EnableEurekaServer
class EurekaServerStarter

fun main(args: Array<String>) {
    runApplication<EurekaServerStarter>(*args)
}
```

application.yml配置
``` Sass
server:
  port: 6600

# eureka相关的配置
eureka:
  instance:
    hostname: localhost
  client:
    # 如果是作为Server来运行的话, 以下两个参数值需为false. 默认为true
    registerWithEureka: false
    fetchRegistry: false
    # 注册中心的地址
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

spring:
  application:
    name: eureka-server
```

启动应用后打开注册中心的地址: http://localhost:6600 可以看到当前注册中心相关信息。
- System Status
系统状态。
- General Info
注册中心的基础信息，包括内存、CPU等等。
- Instance Info
当前实例信息，包括地址，当前状态。
- DS Replicas
当前已注册到Eureka的节点的相关信息。

## EurekaClient

使用上和EurekaServer差不多，需要先启动一个以**@EurekaClient**修饰的SpringBoot应用。

``` Java
@SpringBootApplication
@EnableEurekaClient
class EurekaClientStarter

fun main(args: Array<String>) {
    runApplication<EurekaClientStarter>(*args)
}
```

application.yml配置
``` Sass
server:
  port: 6601

eureka:
  instance:
    hostname: localhost
  client:
    # 上述注册中心的地址
    serviceUrl:
      defaultZone: http://localhost:6600/eureka/

spring:
  application:
    name: eureka-client
```

启动运行后，可在注册中心的页面的DS Replicas部分看到相应名称的注册实例。

## 其他
以上属于单注册中心的配置方式，还可以集群方式配置。  
例如启动两个不同的注册中心，然后再作为服务注册到彼此中。  

示例代码地址: [https://github.com/czb1n/learn-spring-cloud-with-kotlin](https://github.com/czb1n/learn-spring-cloud-with-kotlin)
