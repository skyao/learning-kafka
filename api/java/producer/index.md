# Producer API

Producer API 允许应用程序将数据流发送到 Kafka 集群中的topic。

## maven 依赖

要使用 Producer，可以使用以下 maven 依赖关系：

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>0.10.2.1</version>
</dependency>
```

## Javadoc文档

Producer API 的使用在 javadoc 文档中有简单介绍。

> 以下内容翻译自 http://kafka.apache.org/0102/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html

KafkaProducer 是向 Kafka 集群发布记录的客户端。

producer是线程安全的，跨线程共享单个生产者实例通常比具有多个实例的速度更快。

这是一个简单的例子，使用 producer 发送包含序列号的字符串作为键/值对的记录。