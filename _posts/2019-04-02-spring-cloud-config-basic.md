---
layout: post
author: czb1n
title:  "学习SpringCloud之配置中心Config"
date:   2019-04-02 14:20:00
categories: [Server]
tags: [SpringCloud, Server, Kotlin]
---

## 简介
- SpringCloudConfig是一个集中性、动态的、可拓展的配置服务，并且提供多种存储配置内容的方式，为微服务架构中的其他应用提供配置。

支持存储方式：
> - Git Backend
> - File System Backend
> - Vault Backend
> - JDBC Backend
> - Redis Backend
> - CredHub Backend

配置文件的命名格式：
> /{application}/{profile}[/{label}]  
> /{application}-{profile}.yml  
> /{label}/{application}-{profile}.yml  
> /{application}-{profile}.properties  
> /{label}/{application}-{profile}.properties

application 为配置中的``` spring.config.name ``` 即项目名称。  
profile为``` spring.cloud.config.profile  ```指定的配置类型。  
label为一个git中的可选字段，默认为``` master ```。

> 以下示例均基于SpringCloud的**Greenwich.SR1**版本，且需要依赖到[之前介绍SpringCloud相关的文章](http://www.zbin.tech/tags.html#SpringCloud)

## 基础依赖

``` Html
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-dependencies</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    </dependencies>
```

## ConfigServer
> 开始前，需先启动EurekaServer。

增加依赖
``` Html
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
```

启动时，需要在主类中增加**@EnableConfigServer**注解。
``` Java
@SpringBootApplication
@EnableDiscoveryClient
@EnableConfigServer
class ConfigServerStarter

fun main(args: Array<String>) {
    runApplication<ConfigServerStarter>(*args)
}
```

修改启动配置项 application.yml
``` Sass
server:
  port: 6607

eureka:
  instance:
    hostname: localhost
  client:
    serviceUrl:
      defaultZone: http://localhost:6600/eureka/

spring:
  application:
    name: config-server

  cloud:
    config:
      server:
        # 以git的存储方式进行配置。
        git:
          uri: https://github.com/czb1n/learn-spring-cloud-with-kotlin
          search-paths: config-server
          default-label: master
```
在[配置指定的git库路径下](https://github.com/czb1n/learn-spring-cloud-with-kotlin/tree/master/config-server)中新增``` config-server-dev.properties ``` 文件，内容为```name = czb1n```。  
启动应用访问``` http://localhost:6607/name/dev ```。  
页面返回:
``` Sass
{
	"name": "name",
	"profiles": ["dev"],
	"label": null,
	"version": "af11b65cc6308e46a556229216950472fd4a8fda",
	"state": null,
	"propertySources": []
}
```

## ConfigClient

与Server一样，先增加对应的依赖。
``` Html
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-client</artifactId>
        </dependency>
```
启动类中没有什么特别。
``` Java
@SpringBootApplication
@EnableDiscoveryClient
class ConfigClientStarter

fun main(args: Array<String>) {
    runApplication<ConfigClientStarter>(*args)
}
```
主要是配置文件中需要指定配置中心。
- application.yml:

``` Sass
server:
  port: 6608

spring:
  application:
    name: config-client
```
- bootstrap.yml:

``` Sass
eureka:
  instance:
    hostname: localhost
  client:
    serviceUrl:
      defaultZone: http://localhost:6600/eureka/

spring:
  cloud:
    config:
      discovery:
        enabled: true
        serviceId: config-server
      profile: dev
```
这里有一个问题需要注意，指定配置中心的配置不能在``` application.yml ```中指定，需要在``` bootstrap.yml ```中指定才能生效。  
原因是``` bootstrap.yml ```会先于``` applicaiton.yml ```执行，而这里是因为要指定配置中心，必须要优先于应用的启动配置才能使得配置中心的配置生效。  
> The net result of this behavior is that all client applications that want to consume the Config Server need a bootstrap.yml (or an environment variable) with the server address set in spring.cloud.config.uri (it defaults to "http://localhost:8888").  

在配置中心的git库中新增一个``` config-client-dev.properties ```文件，内容为
``` Sass
name = czb1n
version = 1.0
```
我们在新建一个类，用来测试一下读取配置中心的配置。
``` Java
@RestController
class DemoController {

    @Value("\${name}")
    var name: String? = null

    @Value("\${version}")
    var version: String? = null

    @RequestMapping("/hello")
    fun hello(): String {
        return "hello $name(version $version)."
    }

}
```
启动应用，访问``` http://localhost:6608/hello ```，页面输出``` hello czb1n(version 1.0). ```。  
证明配置中心的配置已生效。

## 其他

示例代码地址: [https://github.com/czb1n/learn-spring-cloud-with-kotlin](https://github.com/czb1n/learn-spring-cloud-with-kotlin)
