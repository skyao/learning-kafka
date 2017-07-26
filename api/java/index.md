# API

Kafka包括四个核心 api：

1. Producer API 允许应用程序将数据流发送到 Kafka 集群中的 topic
2. Consumer API 允许应用程序从 Kafka 群集中的 topic 读取数据流
3. Streams API 允许将 input topic 的数据流转换为 output topic
4. Connect API 允许实现连接器，从某些源系统或应用程序连续拉入数据到 Kafka，或者从Kafka 推送到某个接收器系统或应用程序。

Kafka 通过独立于语言的协议公开其所有功能，该协议具有许多编程语言的客户端。但是，只有Java客户端作为主 Kafka 项目的一部分进行维护，其他客户端作为独立的开源项目。



