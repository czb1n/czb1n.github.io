---
layout: post
author: czb1n
title:  "分布式流处理 Kafka 基础"
date:   2019-04-09 18:02:00
categories: [Server]
tags: [Server, Kafka]
---
## 简介
- [Apache Kafka®](http://kafka.apache.org/) 是一个**分布式流处理平台**，是一种消息中间件。
> 中间件是一种独立的系统软件或服务程序，分布式应用软件借助这种软件在不同的技术之间共享资源。

- 核心概念:
  - Kafka作为一个集群，运行在一台或者多台服务器上。
  - Kafka通过**topic**对存储的流数据进行分类。
  - Kafka中每条记录中都包含**Key**、**Value**和**Timestamp**。

- 核心名词:
  - **Producer**：消息生产者。
  - **Consumer**：消息消费者。
  - **Topic**：消息所属主题，用于区分消息。
  - **Broker**：缓存代理，一个Kafka集群中的服务器称为Broker。

- 核心API:
  - **The Producer API** 允许应用发布流数据到一个或者多个topic。
  - **The Consumer API** 允许应用订阅一个或多个topic，并且对发布给他们的流数据进行处理。
  - **The Streams API** 允许应用作为一个流处理器，消费一个或者多个topic产生的输入流，然后生产一个输出流到一个或多个topic中去，在输入输出流中进行有效的转换。
  - **The Connector API** 允许构建并运行可重用的生产者或者消费者，将topic连接到已存在的应用或者数据系统。比如，连接到一个关系型数据库，捕捉数据表的所有变更内容。
  ![image](http://kafka.apachecn.org/10/images/kafka-apis.png)

- 关键点:
  - Kafka中的topic总是多订阅者模式，一个topic可以拥有一个或者多个消费者来订阅它的数据。
  - 偏移量(即offset)用来唯一的标识分区中每一条记录。
  - Kafka集群所有发布的记录无论他们是否已被消费都会被保留，并可配置保留期限。
  - 偏移量由消费者所控制。
  ![image](http://kafka.apachecn.org/10/images/log_consumer.png)

- Kafka的流行用例: http://kafka.apachecn.org/uses.html

> 注：以上资料包括图片均来自参考[Kafka中文文档](http://kafka.apachecn.org/)。

## 安装使用
> 以下全部以**2.12-2.2.0**版本的Kafka为基础说明。

- 首先[下载](http://kafka.apache.org/downloads)并解压Kafka。
- 由于Kafka依赖于Zookeeper，所以你需要先[下载安装Zookeeper](http://zookeeper.apache.org/releases.html#download)。或者直接使用Kafka包内的Zookeeper脚本来启动一个单节点。  

```python
> bin/zookeeper-server-start.sh config/zookeeper.properties
```

- 指定配置文件启动Kafka。

```python
> bin/kafka-server-start.sh config/server.properties
```

### 发送/接收
- 先创建一个topic。

```python
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic {topic_name}
```

- 启动一个Producer去发送消息。

```python
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic {topic_name}
```

这是命令行处于待输入状态。

- 启动一个Consumer去接收消息。

```python
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic {topic_name} --from-beginning 
```

这时，在producer的命令行中输入一行，在consumer的命令行中就会对应输出一行。

## Java-Client使用

- 引入依赖

```html
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>${kafka-client.version}</version>
</dependency>
```

- Producer

```java
fun main() {
    val props = Properties().apply {
        put("bootstrap.servers", "localhost:9092")
        put("acks", "all")
        put("retries", 0)
        put("batch.size", 16384)
        put("linger.ms", 1)
        put("buffer.memory", 33554432)
        put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer")
        put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")
    }

    KafkaProducer<String, String>(props).use {
        while (true) {
            val message = ProducerRecord("czb1n-topic", "current-time", Instant.now().toString())
            it.send(message)
            Thread.sleep(1000)
        }
    }
}
```

- Consumer

```java
fun main() {
    val props = Properties().apply {
        put("bootstrap.servers", "localhost:9092")
        put("group.id", "test")
        put("enable.auto.commit", "true")
        put("auto.commit.interval.ms", "1000")
        put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    }

    KafkaConsumer<String, String>(props).use {
        it.subscribe(listOf("czb1n-topic"))
        while (true) {
            val messages = it.poll(Duration.ofSeconds(60))
            messages.forEach { m -> println("key = ${m.key()} value = ${m.value()}") }
        }
    }
}
```
启动应用，Producer发送消息时topic不存在会自动创建，可以观察到Consumer会不断输出。
```
...
key = current-time value = 2019-04-10T09:49:12.333Z
key = current-time value = 2019-04-10T09:49:13.337Z
key = current-time value = 2019-04-10T09:49:14.339Z
key = current-time value = 2019-04-10T09:49:15.343Z
key = current-time value = 2019-04-10T09:49:16.347Z
...
```