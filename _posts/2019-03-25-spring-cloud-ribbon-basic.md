---
layout: post
author: czb1n
title:  "学习SpringCloud之负载均衡Ribbon"
date:   2019-03-25 13:24:00
categories: [Server]
tags: [SpringCloud, Server, Kotlin]
---

## 简介
- 什么是负载均衡？  
负载均衡是分布式架构中不可或缺的一个组件，其意义在于通过一定的规则或者算法去将请求分摊到各个服务提供者。

- Ribbon是一个客户端的负载均衡器，它提供一系列让你控制HTTP和TCP客户端的能力。

> 以下示例均基于SpringCloud的**Greenwich.SR1**版本，且需要依赖到[之前介绍Eureka的文章](http://www.zbin.tech/2019/03/21/spring-cloud-eureka-basic.html)

## 基础依赖

``` Html
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>

        <!-- 使用Eureka来配合Ribbon -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    </dependencies>
```

## Ribbon
需要测试Ribbon的功能，需要以下几个步骤。

1. 先运行**Eureka-Server**，启动注册中心。
2. 再运行两个不同端口(6603、6604)的**Eureka-Client**。
需要改一下之前的代码，在 **/hello** 接口中增加端口的输出来区分服务。
``` Java
    @Value("\${server.port}")
    var port: String? = null

    @RequestMapping("/hello")
    fun hello(@RequestParam("name") name: String): String {
        return "response from $port: hello $name."
    }
```

3. 配置启动Ribbon测试应用
因为我们这里需要依赖到Eureka，所以要指定注册中心的地址。  
application.yml  
``` Sass
    server:
      port: 6605

    eureka:
      instance:
        hostname: localhost
      client:
        serviceUrl:
          defaultZone: http://localhost:6600/eureka/

    spring:
      application:
        name: ribbon-client
```
在启动类中，要增加**@EnableDiscoveryClient**注解。
``` Java
    @SpringBootApplication
    @EnableEurekaClient
    @EnableDiscoveryClient
    class RibbonClientStarter

    fun main(args: Array<String>) {
        runApplication<RibbonClientStarter>(*args)
    }
```
启动之后，打开 http://localhost:6600 可以看到现在启动的服务。
```
    EUREKA-CLIENT	UP (2) - 192.168.1.135:eureka-client:6603 , 192.168.1.135:eureka-client:6604
    RIBBON-CLIENT	UP (1) - 192.168.1.135:ribbon-client:6605
```
4. 创建一个Controller去调用**Eureka-Client**的服务。  
先配置创建一个RestTemplate的Bean去负责调用。
``` Java
    @Configuration
    class RestTemplateConfiguration {

        @Bean
        @LoadBalanced
        fun restTemplate(): RestTemplate {
            return RestTemplate()
        }

    }
```
再在Controller中使用RestTemplate。
``` Java
    @RestController
    class DemoController {

        @Autowired
        lateinit var restTemplate: RestTemplate

        @RequestMapping("/hello")
        fun hello(name: String): String? {
            // 其中 EUREKA-CLIENT 为application.yml中的应用名的大写形式，也是注册中心中显示的名字。
            // hello 则为服务提供的接口名。
            return restTemplate.getForObject("http://EUREKA-CLIENT/hello?name=$name", String::class.java)
        }

    }
```

多次访问 http://localhost:6605/hello?name=czb1n，会轮流显示：``` response from 6603: hello czb. ``` 和 ``` response from 6604: hello czb. ```

## 其他

示例代码地址: [https://github.com/czb1n/learn-spring-cloud-with-kotlin](https://github.com/czb1n/learn-spring-cloud-with-kotlin)
