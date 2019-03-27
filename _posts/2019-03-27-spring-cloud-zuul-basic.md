---
layout: post
author: czb1n
title:  "学习SpringCloud之服务网关Zuul"
date:   2019-03-27 18:31:00
categories: [Server]
tags: [SpringCloud, Server, Kotlin]
---

## 简介
- 什么是服务网关？  
实现统一入口，接收所有的请求，并根据定义的规则转发到相应的服务上。  
在此过程中还可以完成系统中一些通用统一的工作，如权限校验，限流等。

- Zuul就是NetFlix提供的一个服务网关，用于实现路由、过滤器等功能。

> Netflix uses Zuul for the following:
> - Authentication
> - Insights
> - Stress Testing
> - Canary Testing
> - Dynamic Routing
> - Service Migration
> - Load Shedding
> - Security
> - Static Response handling
> - Active/Active traffic management

> 以下示例均基于SpringCloud的**Greenwich.SR1**版本，且需要依赖到[之前介绍SpringCloud相关的文章](http://www.zbin.tech/tags.html#SpringCloud)

## 基础依赖

``` Html
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    </dependencies>
```

## Zuul

和其他组件一样，启动一个SpringBoot的应用，启用Zuul需要在启动类中增加**@EnableZuulProxy**注解。
``` Java
@SpringBootApplication
@EnableZuulProxy
class ZuulServerStarter

fun main(args: Array<String>) {
    runApplication<ZuulServerStarter>(*args)
}
```

配置``` application.yml ```也是需要指定Eureka注册中心。
``` Sass
server:
  port: 6606

eureka:
  instance:
    hostname: localhost
  client:
    serviceUrl:
      defaultZone: http://localhost:6600/eureka/

spring:
  application:
    name: zuul-server
```

我们先把[介绍Ribbon文章](http://localhost:4000/2019/03/25/spring-cloud-ribbon-basic.html)中的服务都启动。

接下来我们将Zuul分为路由和过滤器两个部分来说。
- ### 1. 路由  
在```application.yml```增加以下Zuul的配置。
``` Sass
zuul:
  routes:
    # 可以直接指定服务的URL
    api:
      path: /api/**
      url: http://localhost:6603/
    # 也可以指定服务的Id，一般为服务名。
    lbapi:
      path: /lbapi/**
      serviceId: ribbon-client
```
则将```/api```的请求转发到端口为6603的Eureka-Client上。将```/lbapi```的请求转发到Ribbon-Client上。  
启动应用，多次访问```http://localhost:6606/api/hello?name=czb1n```，页面只会显示```response from 6603: hello czb1n.```，
而访问```http://localhost:6606/lbapi/hello?name=czb1n```，页面会轮流显示```response from 6603: hello czb1n.```和```response from 6604: hello czb1n.```。  
通过查看各个服务的日志，也能印证配置路由转发成功。  
增加actuator的配置
``` Sass
management:
  endpoints:
    web:
      exposure:
        include: routes
```
访问```http://localhost:6606/actuator/routes```可以查看路由列表。
``` Sass
{
	"/api/**": "http://localhost:6603/",
	"/lbapi/**": "ribbon-client",
	"/eureka-client/**": "eureka-client",
	"/ribbon-client/**": "ribbon-client"
}
```

- ### 2. 过滤器
    - Zuul有提供一些默认的过滤器，在acturator的include配置项中增加**filters**后，访问```http://localhost:6606/actuator/filters```可以查看。
``` Sass
{
	"error": [{
		"class": "org.springframework.cloud.netflix.zuul.filters.post.SendErrorFilter",
		"order": 0,
		"disabled": false,
		"static": true
	}],
	"post": [{
		"class": "org.springframework.cloud.netflix.zuul.filters.post.SendResponseFilter",
		"order": 1000,
		"disabled": false,
		"static": true
	}],
	"pre": [{
		"class": "org.springframework.cloud.netflix.zuul.filters.pre.DebugFilter",
		"order": 1,
		"disabled": false,
		"static": true
	}, {
		"class": "org.springframework.cloud.netflix.zuul.filters.pre.FormBodyWrapperFilter",
		"order": -1,
		"disabled": false,
		"static": true
	}, {
		"class": "org.springframework.cloud.netflix.zuul.filters.pre.Servlet30WrapperFilter",
		"order": -2,
		"disabled": false,
		"static": true
	}, {
		"class": "org.springframework.cloud.netflix.zuul.filters.pre.ServletDetectionFilter",
		"order": -3,
		"disabled": false,
		"static": true
	}, {
		"class": "org.springframework.cloud.netflix.zuul.filters.pre.PreDecorationFilter",
		"order": 5,
		"disabled": false,
		"static": true
	}],
	"route": [{
		"class": "org.springframework.cloud.netflix.zuul.filters.route.SimpleHostRoutingFilter",
		"order": 100,
		"disabled": false,
		"static": true
	}, {
		"class": "org.springframework.cloud.netflix.zuul.filters.route.RibbonRoutingFilter",
		"order": 10,
		"disabled": false,
		"static": true
	}, {
		"class": "org.springframework.cloud.netflix.zuul.filters.route.SendForwardFilter",
		"order": 500,
		"disabled": false,
		"static": true
	}]
}
```
    - 自定义过滤器可以通过创建继承**ZuulFilter**的Bean来实现。
``` Java
        @Component
        class DemoFilter : ZuulFilter() {
            // 当 shouldFilter() 为true时执行。
            override fun run(): Any? {
                // 这里只是举个例子来模拟权限验证。
                // 当请求参数中的 name 不为 czb1n 时则认为没有访问权限。
                val ctx = RequestContext.getCurrentContext()
                val request = ctx.request
                val response = ctx.response
                if (request.getParameter("name") != "czb1n") {
                    ctx.responseStatusCode = HttpStatus.UNAUTHORIZED.value()
                    // 默认为 true ，为 false 则会过滤此请求，不会分配到路由。
                    ctx.setSendZuulResponse(false)
                    response.writer.write("Sorry, I don't know you.")
                }
                return null
            }

            override fun shouldFilter(): Boolean {
                // 是否需要执行过滤
                return true
            }

            override fun filterType(): String {
                // 过滤器的类型
                return PRE_TYPE
            }

            override fun filterOrder(): Int {
                // 过滤器执行的顺序，由小到大。
                return 0
            }

        }
```
重新启动，访问```http://localhost:6606/api/hello?name=czb1n```结果与原来一致，访问```http://localhost:6606/api/hello?name=anyone```则会显示```Sorry, I don't know you.```
    - 除了一般的过滤器以外，Zuul还包含了**Hystrix**，要为路由配置**fallback**可以通过创建一个继承**FallbackProvider**的Bean来实现。
``` Java
        @Component
        class DemoFallbackProvider : FallbackProvider {

            override fun getRoute(): String {
                // 匹配所有的路由，如果要匹配单个路由则需要返回路由的Id
                return "*"
            }

            override fun fallbackResponse(route: String?, cause: Throwable?): ClientHttpResponse {
                return object : ClientHttpResponse {
                    @Throws(IOException::class)
                    override fun getStatusCode(): HttpStatus {
                        return HttpStatus.OK
                    }

                    @Throws(IOException::class)
                    override fun getRawStatusCode(): Int {
                        return HttpStatus.OK.value()
                    }

                    @Throws(IOException::class)
                    override fun getStatusText(): String {
                        return HttpStatus.OK.reasonPhrase
                    }

                    override fun close() {}

                    @Throws(IOException::class)
                    override fun getBody(): InputStream {
                        return ByteArrayInputStream("fallback response.".toByteArray())
                    }

                    override fun getHeaders(): HttpHeaders {
                        return HttpHeaders()
                    }
                }
            }

        }
```
重启Zuul-Server后，停掉Ribbon-Client服务。  
再访问```http://localhost:6606/lbapi/hello?name=czb1n```页面会显示```fallback response.```。

## 其他

示例代码地址: [https://github.com/czb1n/learn-spring-cloud-with-kotlin](https://github.com/czb1n/learn-spring-cloud-with-kotlin)
