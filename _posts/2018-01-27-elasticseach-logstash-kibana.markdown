---
title:  "基于ELK实现服务器监控告警"
date:   2018-01-27 09:36:49
categories: [Server]
tags: [Server, ElasticSearch, Monitor]
---

## ELK简介

> Elasticsearch

Elasticsearch 是基于 JSON 的分布式搜索和分析引擎，专为实现水平扩展、高可用和管理便捷性而设计。
```
https://www.elastic.co/cn/downloads/elasticsearch
```

> Logstash

Logstash 是动态数据收集管道，拥有可扩展的插件生态系统，能够与 Elasticsearch 产生强大的协同作用。
```
https://www.elastic.co/cn/downloads/logstash
```

> Kibana

Kibana 能够以图表的形式呈现数据，并且具有可扩展的用户界面，供您全方位配置和管理 Elastic Stack。
```
https://www.elastic.co/cn/downloads/kibana
```

## 实现配置

### Logstash配置
Logstash的配置分为3个部分。
- 输入(Input)
- 处理(Filter)
- 输出(Output)

首先说一下输入:
Logstash的输入可以指定多个，而且可以指定收集到的文档的类型。

信息主要分为应用信息和服务器信息两种。

收集应用信息是通过读取各个应用的日志来实现的，所以需要指定文件输入。而服务器信息则是Logstash的命令输入通过执行Collectl(http://collectl.sourceforge.net/)来获取服务器的信息(这里也可以通过Rsyslog来配合Collectl)。
``` json
input {
    file {
        path => '/application/info.log' // 日志文件路径
        type => 'application-log'       // 文档类型
        start_position => beginning     // 指定初始读取位置
    }
    file {
        path => '/application/error.log'
        type => 'application-error-log'
        start_position => beginning
    }
    exec {
        codec => plain {}
        command => 'collectl -c1'       // 需要执行的命令
        interval => 10                  // 执行的间隔
    }
}
```
配置好输入之后就相当于Logstash已经收集到了所有需要的数据，那么只要配置好输出将信息传到Elasticsearch即可。
``` json
elasticsearch {
    hosts => 'localhost'                // Elasticsearch的地址
    index => 'logstash-%{type}'         // Elasticsearch存储的索引名
}
```
可大部分情况下收集到的信息很多是我们不需要或者是格式不符合我们的需求的。这个时候就需要用Filter来过滤和结构化我们收集到的信息。

Filter有很多插件, 如grok、json、ruby...
使用较多的应该是grok，它可以使用正则表达式来匹配你需要的信息，而且Logstash也有了一些预设的Patterns。如果输出的日志较为复杂可以使用一个在线工具(http://grokdebug.herokuapp.com)来调试(Kibana里面的Dev Tools也有这个功能)。

``` json
fitler {
    grok {
        match => {
            "message" => "%{DATA:date} %{DATA:time} %{DATA:log_level} %{DATA:controller} :%{DATA:tracer_id}"
        }
    }
}
```

### Kibana配置

Kibana的配置很简单，只要配置Elasticsearch相关的信息就可以了。

这里主要介绍一下Kibana相关的一些插件。
- x-pack
```
./bin/kibana-plugin install x-pack
```
主要是为了实现用户权限控制和监控功能(Monitoring)，监控Elasticsearch的集群、节点等状态。

- nested-fields-support
根据Kibana的版本号来选择相应版本: https://github.com/ppadovani/KibanaNestedSupportPlugin/releases 安装。
```
./bin/kibana-plugin install "https://github.com/ppadovani/KibanaNestedSupportPlugin/releases/download/6.0.1-1.1.0/nested-fields-support-6.0.1-1.1.0.zip"
```
因为Kibana本身不支持nested类型的聚合查询，而官方一直没有处理这个问题(https://github.com/elastic/kibana/issues/1084)，而这个插件则是为了实现这个功能。安装好之后只要在Kibana中Management->Nested Fields对需要用到这个操作Index启用就可以了。

- sentinl
根据Kibana的版本号来选择相应版本: https://github.com/sirensolutions/sentinl/releases 安装。
```
./bin/kibana-plugin install "https://github.com/sirensolutions/sentinl/releases/download/tag-6.0.0/sentinl-v6.0.0.zip"
```
这个插件主要是为了让Kibana具有告警的能力。安装成功后可以创建相应的观察者(Watchers)定时检测是否符合告警的条件去进行告警。

sentinl的配置主要分为查询条件(Input)、判断条件(Condition)、告警执行的事件(Actions)。

告警执行的事件主要类型有:
1. webhook
2. email
3. report
4. console

### 其他可以加入的优化项

- Kafka
- Beats
