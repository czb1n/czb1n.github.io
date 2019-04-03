---
layout: post
author: czb1n
title:  "学习SpringCloud之服务网关Gateway"
date:   2019-04-02 16:32:00
categories: [Server]
tags: [SpringCloud, Server, Kotlin]
---

## 简介
- **SpringCloudGateway**和**SpringCloudZuul**一样是微服务网关，不过Gateway是SpringCloud官方推出的，而Zuul是Netflix推出的。  
看其他人的一些文章说是Gateway是用于取代Zuul的第二代网关，这个我在官方找不到资料说明。

- 主要术语
    - Route: 路由。
    - Predicate: 断言，即匹配规则。
    - Filter: 过滤器，用于修改请求。

> Route: Route the basic building block of the gateway. It is defined by an ID, a destination URI, a collection of predicates and a collection of filters. A route is matched if aggregate predicate is true.
> 
> Predicate: This is a Java 8 Function Predicate. The input type is a Spring Framework ServerWebExchange. This allows developers to match on anything from the HTTP request, such as headers or parameters.
> 
> Filter: These are instances Spring Framework GatewayFilter constructed in with a specific factory. Here, requests and responses can be modified before or after sending the downstream request.

- 功能实现流程
![image](https://raw.githubusercontent.com/spring-cloud/spring-cloud-gateway/master/docs/src/main/asciidoc/images/spring_cloud_gateway_diagram.png)

> 以下示例均基于SpringCloud的**Greenwich.SR1**版本，且需要依赖到[之前介绍SpringCloud相关的文章](http://www.zbin.tech/tags.html#SpringCloud)

## 基础依赖

``` Html
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>*</artifactId>
                    <groupId>*</groupId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
```

这里有一点要说明一下，因为父模块中有``` spring-boot-starter-web ```的依赖，而**SpringCloudGateway**还不支持``` spring-boot-starter-web ```，所以需要先把依赖排除。  
否则启动会出现以下错误：

``` Java
***************************
APPLICATION FAILED TO START
***************************

Description:

Parameter 0 of method modifyRequestBodyGatewayFilterFactory in org.springframework.cloud.gateway.config.GatewayAutoConfiguration required a bean of type 'org.springframework.http.codec.ServerCodecConfigurer' that could not be found.


Action:

Consider defining a bean of type 'org.springframework.http.codec.ServerCodecConfigurer' in your configuration.
```

## GatewayServer

启动Gateway很简单，只需要一个启动简单的SpringBoot应用就可以。
``` Java
@SpringBootApplication
class GatewayServerStarter

fun main(args: Array<String>) {
    runApplication<GatewayServerStarter>(*args)
}
```

配置Gateway可以使用``` application.yml ```来配置，也可以从代码层面去配置。

如果是用配置文件的方式去配置的话，即用一下的方式去配置路由。
``` Sass
server:
  port: 6609

spring:
  application:
    name: gateway-server

  cloud:
    gateway:
      routes:
        - id: # 路由的标识
        - uri: # 转发的地址
        - predicates: # 指定断句
        - filters: # 指定过滤器
```

用过是代码的方式，则以新建一个**RouteLocator**的Bean来实现。
``` Java
@Configuration
class GatewayConfiguration {

    @Bean
    fun routes(builder: RouteLocatorBuilder): RouteLocator {
        return builder.routes()
            .route {
                it.predicate { p -> p.request.queryParams["name"]?.get(0) == "czb1n" }
                    .filters { f -> f.addRequestHeader("Name", "czb1n") }
                    .uri("http://httpbin.org:80")
            }
            .route {
                it.path("/get")
                    .uri("http://httpbin.org:80")
            }
            .build()
    }

}
```

启动之后，访问``` http://localhost:6609/get ```就会根据第二条转发规则，转发至``` http://httpbin.org:80 ```，页面会显示以下结果。

``` Javascript
{
  "args": {}, 
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3", 
    "Accept-Encoding": "gzip, deflate, br", 
    "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8", 
    "Cache-Control": "max-age=0", 
    "Cookie": "Hm_lvt_a14b0dbc71bff63c2370f65118b12426=1552964642,1553068348,1553132391,1554186102; Hm_lpvt_a14b0dbc71bff63c2370f65118b12426=1554278774", 
    "Forwarded": "proto=http;host=\"localhost:6609\";for=\"0:0:0:0:0:0:0:1:53856\"", 
    "Host": "httpbin.org", 
    "Upgrade-Insecure-Requests": "1", 
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36", 
    "X-Forwarded-Host": "localhost:6609"
  }, 
  "origin": "0:0:0:0:0:0:0:1, ::1", 
  "url": "https://localhost:6609/get"
}
```

访问``` http://localhost:6609/get?name=czb1n ```会匹配第一条规则，也会转发至``` http://httpbin.org:80 ```，但是不会匹配到第二条规则。  
原因是因为规则是从上往下匹配的，只会匹配到符合的第一条规则。  
页面会显示结果。

``` Javascript
{
  "args": {
    "name": "czb1n"
  }, 
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3", 
    "Accept-Encoding": "gzip, deflate, br", 
    "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8", 
    "Cookie": "Hm_lvt_a14b0dbc71bff63c2370f65118b12426=1552964642,1553068348,1553132391,1554186102; Hm_lpvt_a14b0dbc71bff63c2370f65118b12426=1554278774", 
    "Forwarded": "proto=http;host=\"localhost:6609\";for=\"0:0:0:0:0:0:0:1:53856\"", 
    "Host": "httpbin.org", 
    "Name": "czb1n", 
    "Purpose": "prefetch", 
    "Upgrade-Insecure-Requests": "1", 
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86 Safari/537.36", 
    "X-Forwarded-Host": "localhost:6609"
  }, 
  "origin": "0:0:0:0:0:0:0:1, ::1", 
  "url": "https://localhost:6609/get?name=czb1n"
}
```
Filter会在转发请求的**headers**上添加上``` "Name": "czb1n" ```。

## 其他

示例代码地址: [https://github.com/czb1n/learn-spring-cloud-with-kotlin](https://github.com/czb1n/learn-spring-cloud-with-kotlin)
