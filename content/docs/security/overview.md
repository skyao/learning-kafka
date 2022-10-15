---
title: "概述"
linkTitle: "概述"
weight: 10
date: 2022-06-27
description: >
  安全概述
---



> https://kafka.apache.org/documentation/#security_overview



在0.9.0.0版本中，Kafka社区增加了一些功能，这些功能单独或一起使用，可以增加Kafka集群的安全性。目前支持以下安全措施：

1. 使用 SSL 或 SASL 对来自客户端（生产者和消费者）、其他broker和工具的连接进行认证。Kafka支持以下SASL (Simple Authentication and Security Layer/简单认证与安全层)机制：
   - SASL/GSSAPI (Kerberos) - 从版本 0.9.0.0 开始
   - SASL/PLAIN - 从版本 0.10.0.0 开始
   - SASL/SCRAM-SHA-256 和 SASL/SCRAM-SHA-512 - 从版本 0.10.2.0 开始
   - SASL/OAUTHBEARER - 从版本 2.0 开始
2. 验证从 broker 到 ZooKeeper 的连接
3. 使用 SSL 对 broker 和客户端之间、broker之间或 broker 和工具之间传输的数据进行加密（注意，启用 SSL 后会有性能下降，其程度取决于 CPU 类型和 JVM 实现）。
4. 对客户的读/写操作进行授权
5. 授权是可插拔的，支持与外部授权服务的集成

值得注意的是，安全是可选的--支持非安全的集群，以及认证的、非认证的、加密的和非加密的客户端的混合。下面的指南解释了如何配置和使用客户端和 broker 的安全功能。
