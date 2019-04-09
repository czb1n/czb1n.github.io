---
layout: post
author: czb1n
title:  "学习SpringCloud之服务注册与发现Consul"
date:   2019-04-09 09:10:00
categories: [Server]
tags: [SpringCloud, Server, Kotlin]
---

## 简介
- **SpringCloudConsul**和**SpringCloudEureka**一样是作用于微服务架构中的服务治理。  
[由于Eureka已经停止维护](https://github.com/Netflix/eureka/wiki)，Consul是一个很好的替代品。

- 除了服务治理以外，Consul还提供一个简易的键/值储存，这可以用于作为动态配置等等。

> 以下示例均基于SpringCloud的**Greenwich.SR1**版本，Consul版本为**v1.4.4**，且需要依赖到[之前介绍SpringCloud相关的文章](http://www.zbin.tech/tags.html#SpringCloud)

## 基础依赖

``` Html
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
    </dependencies>
```

## ConsulServer

使用前需先安装、运行Consul的服务端，具体可以参考[官网的安装教程](https://learn.hashicorp.com/consul/getting-started/install.html)。  
简单说一下流程：
1. 在[下载页面](https://www.consul.io/downloads.html)下载自己环境适用的版本。
2. 执行以下命令以开发环境启动Consul的服务端。  
``` JavaScript
consul agent -dev -bind=127.0.0.1 -client=0.0.0.0
```

Consul的默认端口是**8500**，访问``` http://localhost:8500 ```可以看到Consul的控制面板。

- Services：显示所有服务的信息、状态。
- Nodes：显示所有的节点。
- Key/Value：显示所有的键值对。
    - ```http://127.0.0.1:8500/v1/kv/?recurse``` 列出所有的键值对。
    - ```http://127.0.0.1:8500/v1/kv/{key}``` 获取单个键的值。
- ACL：访问控制列表配置。
- Intentions：服务连接配置。

## ConsulClient

配置文件```application.yml```中，指定ConsulServer的地址。
``` Sass
server:
  port: 6610

spring:
  application:
    name: consul-client
  cloud:
    consul:
      host: localhost
      port: 8500
```
这里官方文档中提到需要注意一点。
> If you use Spring Cloud Consul Config, the above values will need to be placed in bootstrap.yml instead of application.yml.

像Eureka一样，启动一个简单的**SpringBoot**应用，并作一个简单的controller去测试。
``` Java
@SpringBootApplication
class ConsulClientStarter

fun main(args: Array<String>) {
    runApplication<ConsulClientStarter>(*args)
}
```

``` Java
@RestController
class DemoController {

    @Value("\${server.port}")
    var port: String? = null

    @RequestMapping("/hello")
    fun hello(@RequestParam("name") name: String): String {
        return "response from $port: hello $name."
    }

}
```

启动两个端口分别为**6610**和**6611**的**ConsulClient**，再新建一个模块，以Ribbon的方式去调用服务，测试一下服务发现。

``` Sass
server:
  port: 6612

spring:
  application:
    name: consul-ribbon-client
  cloud:
    consul:
      host: localhost
      port: 8500
```

``` Java
@SpringBootApplication
@EnableDiscoveryClient
class ConsulRibbonClientStarter

fun main(args: Array<String>) {
    runApplication<ConsulRibbonClientStarter>(*args)
}
```

``` Java
@RestController
class DemoController {

    @Autowired
    lateinit var restTemplate: RestTemplate

    @RequestMapping("/hello")
    fun hello(@RequestParam("name") name: String): String? {
        return restTemplate.getForObject("http://CONSUL-CLIENT/hello?name=$name", String::class.java)
    }

}
```

启动好各个服务后，再看看Consul的控制面板中的Services可以看到多了两个服务。  
其中ConsulClient有两个实例，ConsulRibbonClient有一个实例。  
每个服务的信息中，有一个项为```Health Checks```健康检查。  
这个健康检查是Consul定时访问一个各个服务提供的检查接口。  
例如其中一个ConsulClient提供的接口为```http://localhost:6610/actuator/health```，返回```{"status":"UP"}```。  
每个服务也可以单独指定健康检查的接口。

多次访问```http://localhost:6612/hello?name=czb1n```，页面轮流返回```response from 6610: hello czb1n.```和```response from 6611: hello czb1n.```。

## 其他

示例代码地址: [https://github.com/czb1n/learn-spring-cloud-with-kotlin](https://github.com/czb1n/learn-spring-cloud-with-kotlin)
