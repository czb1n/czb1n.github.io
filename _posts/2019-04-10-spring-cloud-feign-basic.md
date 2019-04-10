---
layout: post
author: czb1n
title:  "学习SpringCloud之服务调用Feign"
date:   2019-04-10 10:13:00
categories: [Server]
tags: [SpringCloud, Server, Kotlin]
---

## 简介
- **SpringCloudFeign**是的作用是微服务间实现声明式的调用。同时还整合了**Ribbon**和**Hystrix**的功能。

- 声明式调用的好处在于，避免了像之前介绍**Ribbon**时使用**RestTemplate**调用服务那样，需要拼接请求URL，包括请求参数。  
这样不但减少出错的机会，代码上也比较整洁好阅读和理解。

> 以下示例均基于SpringCloud的**Greenwich.SR1**版本，且需要依赖到[之前介绍SpringCloud相关的文章](http://www.zbin.tech/tags.html#SpringCloud)

## 基础依赖

```html
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
    </dependencies>
```

因为这次使用Consul作为服务治理，所以也需要引入Consul的依赖。

## FeignCommon

这个模块主要是为了声明一些服务的接口，让Provider和Consumer去引入使用。

```java
@FeignClient("feign-provider", fallbackFactory = HelloServiceFallbackFactory::class)
interface HelloService {

    @RequestMapping("/hello")
    fun hello(@RequestParam("name") name: String): String

}
```
首先我们使用**@FeignClient**注解声明一个服务接口。  
第一个参数为提供实现的服务名称。  
第二个参数则是为了实现**Hystrix**的熔断功能。

这需要提供一个实现了这个接口的类来提供Fallback结果。
```java
@Component
class HelloServiceFallback : HelloService {

    override fun hello(name: String): String {
        return "fallback function: hello $name."
    }

}
```
留意到服务接口定义中，是使用了工厂模式，所以还需要一个定义一个工厂类。  
使用时也可以不用工厂模式。
```java
@Component
class HelloServiceFallbackFactory(val helloServiceFallback: HelloServiceFallback) : FallbackFactory<HelloService> {

    override fun create(throwable: Throwable?): HelloService {
        // 当出现错误时，打印错误信息。
        println("hello service fallback reason : ${throwable?.message ?: "unknown error."}")
        return helloServiceFallback
    }

}
```

## FeignProvider

负责实现服务接口功能，先配置Consul地址并打开Hystrix功能。
```sass
server:
  port: 6613

spring:
  application:
    name: feign-provider
  cloud:
    consul:
      host: localhost
      port: 8500

feign:
  hystrix:
    enabled: true
```
留意到``` spring.application.name ```和服务接口定义时@FeignClient指定的服务名是一致的。

```java
@RestController
class HelloServiceImpl : HelloService {

    @Value("\${server.port}")
    var port: String? = null

    override fun hello(name: String): String {
        return "response from $port: hello $name."
    }

}
```
实现中不用再去写**@RequestMapping**，只需要重写接口就可以，因为在服务定义的时候已经指定了。

接下来只要在启动类中增加**@EnableFeignClients**注解启动就可以了。
```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
class FeignProviderStarter

fun main(args: Array<String>) {
    runApplication<FeignProviderStarter>(*args)
}
```

## FeignConsumer

负责模拟服务调用，同样先配置好应用信息。
```sass
server:
  port: 6614

spring:
  application:
    name: feign-customer
  cloud:
    consul:
      host: localhost
      port: 8500

feign:
  hystrix:
    enabled: true
```

然后创建一个Controller去调用。
```java
@RestController
class DemoController {

    @Autowired
    lateinit var helloService: HelloService

    @RequestMapping("/hello")
    fun hello(@RequestParam("name") name: String): String {
        return helloService.hello(name)
    }

}
```
> 这里有一点要注意，由于helloService这个Bean是启动时注入的，编译器无办法知道导致了会报错。  
> 这个错误可以忽略，不影响启动和使用。

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
class FeignCustomerStarter

fun main(args: Array<String>) {
    runApplication<FeignCustomerStarter>(*args)
}
```

启动应用，访问``` http://localhost:6614/hello?name=czb1n ```，如果**feign-provider**服务状态正常时，会显示``` response from 6613: hello czb1n. ```。否则会返回``` fallback function: hello czb1n. ```

## 其他

示例代码地址: [https://github.com/czb1n/learn-spring-cloud-with-kotlin](https://github.com/czb1n/learn-spring-cloud-with-kotlin)
