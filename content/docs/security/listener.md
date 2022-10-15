---
title: "Listener"
linkTitle: "Listener"
weight: 20
date: 2022-06-27
description: >
  监听器配置
---

> https://kafka.apache.org/documentation/#listener_configuration

为了保证 Kafka 集群的安全，有必要保证用于与服务器通信的通道的安全。每个服务器必须定义一组监听器，用来接收来自客户端以及其他服务器的请求。每个监听器可以被配置为使用各种机制来验证客户端，并确保服务器和客户端之间的通信是安全的。本节提供了一个配置监听器的入门知识。

Kafka 服务器支持监听多个端口的连接。这是通过服务器配置中的 `listeners` 属性来配置的，它接受一个用逗号分隔的监听器列表来启用。每个服务器上必须至少定义一个监听器。 `listeners` 中定义的每个监听器的格式如下：

```http
{LISTENER_NAME}://{hostname}:{port}
```

`LISTENER_NAME`通常是一个描述性的名字，定义了监听器的用途。例如，许多配置为客户端流量使用单独的监听器，所以他们可能在配置中把相应的监听器称为`CLIENT':

```
listeners=CLIENT://localhost:9092
```

每个监听器的安全协议在一个单独的配置中定义：`listener.security.protocol.map`。该值是一个逗号分隔的列表，列出了映射到其安全协议的每个监听器。例如，下面值配置指定 `CLIENT` 监听器将使用 SSL，而`BROKER` 监听器将使用明文（plaintext）。

```properties
listener.security.protocol.map=CLIENT:SSL,BROKER:PLAINTEXT
```

下面给出了安全协议的可能选项：

1. PLAINTEXT
2. SSL
3. SASL_PLAINTEXT
4. SASL_SSL

明文（PLAINTEXT）协议不提供安全性，不需要任何额外的配置。在下面的章节中，本文将介绍如何配置其余的协议。

如果每个需要的监听器使用单独的安全协议，也可以在监听器中使用安全协议名称作为监听器名称。使用上面的例子，我们可以使用下面的定义跳过 CLIENT 和 BROKER 监听器的定义：

```http
listeners=SSL://localhost:9092,PLAINTEXT://localhost:9093
```

然而，我们建议用户为监听器提供明确的名称，因为它使每个监听器的预期用途更加清晰。

在这个列表中的监听器中，可以通过将 `inter.broker.listener.name` 配置设置为监听器的名称，来声明监听器用于 broker 间的通信。broker 间监听器的主要目的是分区复制。如果没有定义，那么 broker 间的监听器由 `security.inter.broker.protocol` 定义的安全协议决定，该协议默认为`PLAINTEXT`。

对于依赖 Zookeeper 存储集群元数据的传统集群，可以声明一个单独的监听器，用于从活动控制器到 broker 的元数据传播。这是由 `control.plane.listener.name` 定义的。当活动控制器需要向集群中的 broker 推送元数据更新时，它将使用这个监听器。使用控制平面监听器的好处是，它使用一个单独的处理线程，这使得应用程序流量不太可能阻碍元数据变化的及时传播（如分区领导和ISR更新）。注意，默认值是 null，这意味着控制器将使用由 `inter.broker.listener` 定义的相同监听器。

控制器接收来自其他控制器和 broker 的请求。由于这个原因，即使一个服务器没有启用控制器角色（即它只是一个 broker），它仍然必须定义控制器监听器以及配置它所需的任何安全属性。例如，我们可以在一个独立的 broker 上使用以下配置：

```properties
process.roles=broker
listeners=BROKER://localhost:9092
inter.broker.listener.name=BROKER
controller.quorum.voters=0@localhost:9093
controller.listener.names=CONTROLLER
listener.security.protocol.map=BROKER:SASL_SSL,CONTROLLER:SASL_SSL
```

在这个例子中，控制器监听器仍然被配置为使用 `SASL_SSL` 安全协议，但它不包括在 `listeners` 中，因为 broker 没有暴露控制器监听器本身。在这种情况下，将使用的端口来自 `controller.quorum.voters` 配置，它定义了完整的控制器列表。

对于同时启用了 broker 和控制器角色的 KRaft 服务器，配置是类似的。唯一的区别是，控制器监听器必须包含在监听器中:

```properties
process.roles=broker,controller
listeners=BROKER://localhost:9092,CONTROLLER://localhost:9093
inter.broker.listener.name=BROKER
controller.quorum.voters=0@localhost:9093
controller.listener.names=CONTROLLER
listener.security.protocol.map=BROKER:SASL_SSL,CONTROLLER:SASL_SSL
```

在 controller.quorum.voters 中定义的端口必须与暴露的控制器监听器之一完全匹配。例如，这里的 CONTROLLER 监听器被绑定到端口9093。由 controller.quorum.voters 定义的连接字符串也必须使用端口 9093，就像这里一样。

控制器将接受由 controller.listener.names 定义的所有监听器的请求。通常情况下，只有一个控制器监听器，但也可以有更多。例如，这提供了一种方法，通过集群的滚动，将活动的监听器从一个端口或安全协议改为另一个（一个滚动暴露新的监听器，一个滚动删除旧的监听器）。当定义了多个控制器监听器时，列表中的第一个监听器将被用于出站请求。

在 Kafka 中，传统的做法是为客户端使用单独的监听器。这使得集群间的监听器可以在网络层面上被隔离。在 KRaft 的控制器监听器的情况下，监听器应该是隔离的，因为客户端无论如何都不会与它一起工作。客户端应该连接到配置在代理上的任何其他监听器。任何与控制器绑定的请求将被转发，如下所述。

在下一节中，本文将介绍如何在监听器上启用 SSL 以进行加密和验证。随后的部分将介绍使用 SASL 的额外认证机制。
