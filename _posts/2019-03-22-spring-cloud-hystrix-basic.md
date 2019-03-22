---
layout: post
author: czb1n
title:  "学习SpringCloud之断路器Hystrix"
date:   2019-03-22 13:16:00
categories: [Server]
tags: [SpringCloud, Server, Kotlin]
---

## 简介
- 什么是断路器？  
断路器就是为了解决微服务架构中的“雪崩”现象，即某个服务出现问题会导致其他服务阻塞，严重最终会导致服务器瘫痪。  
当服务出现问题是，断路器会负责断开这个该服务的依赖，以防止问题蔓延，保护整体服务。

- **Hystrix**也是**SpringCloudNetflix**微服务套件中的一个组件，作为断路器的角色。

![image](https://raw.githubusercontent.com/spring-cloud/spring-cloud-netflix/master/docs/src/main/asciidoc/images/HystrixFallback.png)

> 以下示例均基于SpringCloud的**Greenwich.SR1**版本。

## 基础依赖

``` Html
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
    </dependencies>
```

## Hystrix

以**@EnableHystrix**修饰启动一个SpringBoot应用。
``` Java
@SpringBootApplication
@EnableHystrix
class HystrixClientStarter

fun main(args: Array<String>) {
    runApplication<HystrixClientStarter>(*args)
}
```

application.yml配置
``` Sass
server:
  port: 6602

spring:
  application:
    name: hystrix-client
```

再写一个简单的Controller来试一下效果。
``` Java
@RestController
class DemoController {

    @RequestMapping("hello")
    @HystrixCommand(fallbackMethod = "fallback")
    fun hello(@RequestParam("name") name: String): String {
        throw Exception("test exception")               // throw test exception here
        return "hello $name."
    }


    fun fallback(name: String): String {
        return "fallback function: hello $name."
    }
}
```

用**@HystrixCommand**注解来为一个方法增加熔断的能力。只需要定义一个和目标方法(上述例子为**hello()**)结构一致的方法，在注解中传入即可。  
为了测试，目标方法中故意抛了一个异常来模拟微服务异常。  
访问 http://localhost:6602/hello?name=czb1n 会显示：
```
fallback function: hello czb1n.
```

## HystrixDashboard

Hystrix还提供了一个数据指标面板工具，HystrixDashboard。不过这个工具没有在Hystrix包里，需要另外导入。
``` Html
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
    </dependency>
```

启动类中要增加**@EnableHystrixDashboard**注解来启动面板。  
面板中的指标数据来自于SpringBoot的**actuator**。  
所以要加入actuator的依赖的同时，还要修改application.yml来暴露Hystrix相关的节点指标。
``` Sass
management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream
```

启动访问http://localhost:6602/actuator/hystrix.stream 就可以看到指标数据。  
访问 http://localhost:6602/hystrix 就能打开HystrixDashboard的界面。输入上述的stream就能监控对应的指标。

![image](https://raw.githubusercontent.com/spring-cloud/spring-cloud-netflix/master/docs/src/main/asciidoc/images/Hystrix.png)

## 其他
图片均来自[SpringCloudNetflix官方文档](https://cloud.spring.io/spring-cloud-netflix/spring-cloud-netflix.html#_circuit_breaker_hystrix_clients)

示例代码地址: [https://github.com/czb1n/learn-spring-cloud-with-kotlin](https://github.com/czb1n/learn-spring-cloud-with-kotlin)
