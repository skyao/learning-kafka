---
title: "授权"
linkTitle: "授权"
weight: 50
date: 2022-06-27
description: >
  授权和ACL
---

> https://kafka.apache.org/documentation/#security_authz



Kafka 提供了一个可插拔的授权框架，它是通过服务器配置中的 `authorizer.class.name` 属性配置的。配置的实现必须扩展 `org.apache.kafka.server.authorizer.Authorizer` 。Kafka 提供了默认的实现，将ACL存储在集群元数据中（Zookeeper或KRaft元数据日志）。对于基于Zookeeper的集群，所提供的实现是这样配置的。

```properties
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
```

Kafka acls 的一般格式是 "Principal P是 [Allowed/Denied] Operation O From Host H on any Resource R matching ResourcePattern RP"。你可以在 [KIP-11 ](https://cwiki.apache.org/confluence/display/KAFKA/KIP-11+-+Authorization+Interface)中阅读更多关于acl结构的信息，在[KIP-290](https://cwiki.apache.org/confluence/display/KAFKA/KIP-290%3A+Support+for+Prefixed+ACLs)中阅读资源模式。为了添加、删除或列出acls，你可以使用Kafka authorizer CLI。默认情况下，如果没有ResourcePatterns匹配特定的资源R，那么R就没有相关的acls，因此除了超级用户之外，其他任何人都不允许访问R。如果你想改变这种行为，你可以在 server.properties 中包含以下内容。

```properties
allow.everyone.if.no.acl.found=true
```

也可以像下面这样在 server.properties 中添加超级用户（注意分界符是分号，因为SSL用户名可能包含逗号）。默认的PrincipalType字符串 "User" 是区分大小写的。

```properties
super.users=User:Bob;User:Alice
```

TBD
