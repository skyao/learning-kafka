---
title: "SASL"
linkTitle: "SASL"
weight: 40
date: 2022-06-27
description: >
  使用SASL进行认证
---

> https://kafka.apache.org/documentation/#security_sasl

## 1. JAAS 配置

Kafka使用 Java认证和授权服务（Java Authentication and Authorization Service / JAAS）进行SASL配置。

### 用于Kafka broker的JAAS配置

`KafkaServer `是每个 KafkaServer/Broker 使用的 JAAS 文件中的部分名称。该部分为 broker 提供 SASL 配置选项，包括 broker 为 broker 之间的通信进行的任何SASL客户端连接。如果多个监听器被配置为使用SASL，该部分的名称可以以小写的监听器名称为前缀，后面跟一个句号，例如：`sasl_ssl.KafkaServer`。

`client` 部分用于验证与 zookeeper 的 SASL 连接。它还允许 broker 在 zookeeper 节点上设置 SASL ACL，锁定这些节点，以便只有 broker 可以修改它。有必要在所有的 broker 中使用相同的 principal 名称。如果你想使用Client以外的部分名称，请将系统属性 `zookeeper.sasl.clientconfig` 设置为合适的名称（*如*，`-Dzookeeper.sasl.clientconfig=ZkClient`）。

ZooKeeper默认使用 "zookeeper" 作为服务名称。如果你想改变这一点，请将系统属性 `zookeeper.sasl.client.username`设置为适当的名称（*例如*，`-Dzookeeper.sasl.client.username=zk`）。

broker 也可以使用 broker 配置属性 `sasl.jaas.config` 来配置 JAAS。该属性名称必须以包括 SASL 机制的监听器前缀，即`listener.name.{listenerName}.{saslMechanism}.sasl.jaas.config`。配置值中只能指定一个登录模块。如果在一个监听器上配置了多个机制，必须使用监听器和机制前缀为每个机制提供配置。例如:

```properties
listener.name.sasl_ssl.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="admin" \
    password="admin-secret";
listener.name.sasl_ssl.plain.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
    username="admin" \
    password="admin-secret" \
    user_admin="admin-secret" \
    user_alice="alice-secret";
```

如果JAAS配置是在不同层次上定义的，所使用的优先顺序是：

- broker 配置属性 `listener.name.{listenerName}.{saslMechanism}.sasl.jaas.config`
- 静态JAAS配置的`{listenerName}.KafkaServer` 部分
- 静态JAAS配置的`KafkaServer` 部分

请注意，ZooKeeper JAAS配置只能使用静态JAAS配置进行配置。

关于代理配置的例子，请参见[GSSAPI (Kerberos)](https://kafka.apache.org/documentation/#security_sasl_kerberos_brokerconfig)、[PLAIN](https://kafka.apache.org/documentation/#security_sasl_plain_brokerconfig)、[SCRAM](https://kafka.apache.org/documentation/#security_sasl_scram_brokerconfig) 或 [OAUTHBEARER](https://kafka.apache.org/documentation/#security_sasl_oauthbearer_brokerconfig) 。

### 为Kafka客户端配置JAAS

客户端可以使用客户端配置属性[sasl.jaas.config](https://kafka.apache.org/documentation/#security_client_dynamicjaas)或使用类似于经纪人的[静态JAAS配置文件](https://kafka.apache.org/documentation/#security_client_staticjaas)来配置JAAS:

1. ###### 使用客户端配置属性进行JAAS配置

   客户端可以将 JAAS 配置指定为生产者或消费者属性，而不需要创建一个物理配置文件。这种模式也使同一JVM内的不同生产者和消费者能够通过为每个客户指定不同的属性来使用不同的凭证。如果同时指定静态JAAS配置系统性`java.security.auth.login.config`和客户端属性`sasl.jaas.config`，将使用客户端属性。

   关于配置的例子，请参见[GSSAPI (Kerberos)](https://kafka.apache.org/documentation/#security_sasl_kerberos_clientconfig)、[PLAIN](https://kafka.apache.org/documentation/#security_sasl_plain_clientconfig)、[SCRAM](https://kafka.apache.org/documentation/#security_sasl_scram_clientconfig)或[OAUTHBEARER](https://kafka.apache.org/documentation/#security_sasl_oauthbearer_clientconfig)。

2. ###### 使用静态配置文件进行JAAS配置

   使用静态JAAS配置文件在客户端配置SASL认证。

   1. 添加 JAAS 配置文件，其中有名为 KafkaClient 的客户端登录部分。按照设置GSSAPI（Kerberos）、PLAIN、SCRAM或OAUTHBEARER的例子所述，在 KafkaClient 中为所选机制配置登录模块。例如，GSSAPI 凭证可以配置为:

      ```properties
   KafkaClient {
          com.sun.security.auth.module.Krb5LoginModule required
       useKeyTab=true
          storeKey=true
       keyTab="/etc/security/keytabs/kafka_client.keytab"
          principal="kafka-client-1@EXAMPLE.COM";
   };
      ```

   2. 将JAAS配置文件的位置作为JVM参数传递给每个客户JVM。比如说:

      ```bash
      -Djava.security.auth.login.config=/etc/kafka/kafka_client_jaas.conf
      ```

## 2. SASL 配置

SASL 可以使用 PLAINTEXT 或 SSL 作为传输层，分别使用安全协议 SASL_PLAINTEXT 或 SASL_SSL。如果使用 SASL_SSL，那么还必须配置 SSL。

### SASL 机制

Kafka支持以下 SASL 机制：

- [GSSAPI](https://kafka.apache.org/documentation/#security_sasl_kerberos) (Kerberos)
- [PLAIN](https://kafka.apache.org/documentation/#security_sasl_plain)
- [SCRAM-SHA-256](https://kafka.apache.org/documentation/#security_sasl_scram)
- [SCRAM-SHA-512](https://kafka.apache.org/documentation/#security_sasl_scram)
- [OAUTHBEARER](https://kafka.apache.org/documentation/#security_sasl_oauthbearer)

### Kafka broker 的 SASL 配置

1. 在 server.properties 中配置 SASL 端口，在 listeners 参数中至少添加 SASL_PLAINTEXT 或 SASL_SSL 之一，该参数包含一个或多个逗号分隔的值。

   ```properties
   listeners=SASL_PLAINTEXT://host.name:port
   ```

   如果你只配置一个 SASL 端口（或者你想让 Kafka broker 使用 SASL 互相认证），那么请确保你为 broker 之间的通信设置相同的SASL协议。

   ```properties
   security.inter.broker.protocol=SASL_PLAINTEXT (or SASL_SSL)
   ```

2. 选择一个或多个 [支持的机制](https://kafka.apache.org/documentation/#security_sasl_mechanism) 在 broker 中启用，并按照步骤为该机制配置SASL。要在代理中启用多个机制，请按照步骤[这里](https://kafka.apache.org/documentation/#security_sasl_multimechanism)。

### Kafka 客户端的 SASL 配置

SASL 认证只支持新的 Java Kafka 生产者和消费者，旧的API不被支持。

要在客户端配置 SASL 认证，请选择 broker 中为客户端认证启用的 SASL[机制]()，并按照步骤为所选机制配置SASL。

## 3. 使用SASL/Kerberos进行认证

### 先决条件

1. **Kerberos**

   如果你的组织已经在使用Kerberos服务器（例如，通过使用Active Directory），就没有必要为 Kafka 安装一个新的服务器。否则你需要安装一个，你的Linux供应商可能有 Kerberos 的软件包，以及如何安装和配置的简短指南（[Ubuntu](https://help.ubuntu.com/community/Kerberos), [Redhat](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Managing_Smart_Cards/installing-kerberos.html)）。注意，如果你使用的是Oracle Java，你将需要为你的Java版本下载JCE策略文件，并将它们复制到$JAVA_HOME/jre/lib/security。

1. 创建Kerberos原则

   如果你使用的是组织的Kerberos或Active Directory服务器，请向你的Kerberos管理员要一个principal，用于集群中的每个Kafka broker 和每个将用Kerberos认证（通过客户端和工具）访问Kafka的操作系统用户。

   如果你已经安装了你自己的Kerberos，你将需要使用以下命令自己创建这些 principals。

   ```bash
   > sudo /usr/sbin/kadmin.local -q 'addprinc -randkey kafka/{hostname}@{REALM}'
   > sudo /usr/sbin/kadmin.local -q "ktadd -k /etc/security/keytabs/{keytabname}.keytab kafka/{hostname}@{REALM}"
   ```

2. **确保所有的主机都可以用主机名到达**--这是Kerberos的要求，你的所有主机都可以用它们的FQDNs来解析。

### 配置 kafka broker

1. 在每个 Kafka broker 的配置目录下添加一个与下面类似的经过适当修改的 JAAS 文件，在这个例子中我们称之为kafka_server_jaas.conf（注意，每个 broker 都应该有自己的keytab）。

   ```properties
    KafkaServer {
      com.sun.security.auth.module.Krb5LoginModule required
      useKeyTab=true
      storeKey=true
      keyTab="/etc/security/keytabs/kafka_server.keytab"
      principal="kafka/kafka1.hostname.com@EXAMPLE.COM";
    };
   
   // Zookeeper client authentication
   Client {
      com.sun.security.auth.module.Krb5LoginModule required
      useKeyTab=true
      storeKey=true
      keyTab="/etc/security/keytabs/kafka_server.keytab"
      principal="kafka/kafka1.hostname.com@EXAMPLE.COM";
   };
   ```

   JAAS 文件中的 KafkaServer 部分告诉 broker 要使用哪个 prinsipal ，以及存储该 principal 的 keytab 的位置。它允许 broker 使用本节中指定的 keytab 登录。关于Zookeeper SASL配置的更多细节，请参见注释。

2. 将 JAAS 和可选的 krb5 文件位置作为 JVM 参数传递给每 个Kafka broker（更多细节见这里）。

   ```bash
   -Djava.security.krb5.conf=/etc/kafka/krb5.conf
   -Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf
   ```

3. 确保在 JAAS 文件中配置的 keytabs 是可以被启动 kafka broker 的操作系统用户读取的。

4. 在 server.properties 中配置SASL端口和SASL机制，如这里所述. 例如：

   ```properties
   listeners=SASL_PLAINTEXT://host.name:port
   security.inter.broker.protocol=SASL_PLAINTEXT
   sasl.mechanism.inter.broker.protocol=GSSAPI
   sasl.enabled.mechanisms=GSSAPI
   ```

   我们还必须在 server.properties 中配置服务名称，它应该与 kafka broker 的 principal 名称相匹配。在上面的例子中，principal 是 "kafka/kafka1.hostname.com@EXAMPLE.com"，所以:

   ```properties
   sasl.kerberos.service.name=kafka
   ```

### 配置 Kafka Clients

要在客户端配置SASL认证:

1. 客户端（生产者、消费者、连接工作者等）将用自己的 broker（通常与运行客户端的用户同名）对集群进行认证，因此要根据需要获得或创建这些 principal。然后为每个客户端配置 JAAS 配置属性。一个 JVM 内的不同客户端可以通过指定不同的 principal，作为不同的用户运行。producer.properties 或 consumer.properties中 的属性 `sasl.jaas.config` 描述了生产者和消费者等客户端如何连接到 Kafka Broker。下面是一个使用keytab的客户端的配置示例（推荐用于长期运行的进程）。

   ```properties
   sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
       useKeyTab=true \
       storeKey=true  \
       keyTab="/etc/security/keytabs/kafka_client.keytab" \
       principal="kafka-client-1@EXAMPLE.COM";
   ```

   对于 kafka-console-consumer 或 kafka-console-producer 这样的命令行工具，kinit 可以和 "useTicketCache=true" 一起使用，如：

   ```properties
   sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
       useTicketCache=true;
   ```

   客户端的 JAAS 配置也可以指定为 JVM 参数，类似于这里所说的 broker。客户端使用名为 KafkaClient 的登录部分。这个选项只允许一个JVM的所有客户端连接的用户。

2. 确保在 JAAS 配置中配置的 keytabs 是可以被启动 kafka 客户端的操作系统用户读取的。

3. 可以选择将krb5文件的位置作为JVM参数传递给每个客户端JVM（更多细节见这里）。

   1. ```bash
      -Djava.security.krb5.conf=/etc/kafka/krb5.conf
      ```

4. 在 producer.properties 或 consumer.properties 中配置以下属性:

   ```properties
   security.protocol=SASL_PLAINTEXT (or SASL_SSL)
   sasl.mechanism=GSSAPI
   sasl.kerberos.service.name=kafka
   ```

## 4. 使用 SASL/PLAIN 进行认证

SASL/PLAIN 是一种简单的用户名/密码认证机制，通常与 TLS 一起用于加密以实现安全认证。Kafka 支持 SASL/PLAIN 的默认实现，它可以被扩展为生产使用，如这里所描述的。

在 principal.builder.class 的默认实现下，username 被用作配置 ACL 等的认证 Principal。

### 配置 Kafka Brokers

1. 在每个Kafka broker 的配置目录下添加一个与下面类似的经过适当修改的JAAS文件，本例中我们称之为 kafka_server_jaas.conf：

   ```properties
   KafkaServer {
       org.apache.kafka.common.security.plain.PlainLoginModule required
       username="admin"
       password="admin-secret"
       user_admin="admin-secret"
       user_alice="alice-secret";
   };
   ```

   这个配置定义了两个用户（admin和alice）。KafkaServer 部分的属性 username 和 password 被 broker 用来启动与其他 broker 的连接。在这个例子中，admin 是用于 broker 之间通信的用户。一组属性 user_userName 定义了所有连接到 broker 的用户的密码， broker 使用这些属性验证所有的客户端连接，包括来自其他 broker 的连接。

2. 将 JAAS 配置文件的位置作为 JVM 参数传递给每个 Kafka broker

   ```bash
   -Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf
   ```

3. 在 server.properties 中配置 SASL 端口和 SASL机制，如上所述。比如说。

   ```properties
   listeners=SASL_SSL://host.name:port
   security.inter.broker.protocol=SASL_SSL
   sasl.mechanism.inter.broker.protocol=PLAIN
   sasl.enabled.mechanisms=PLAIN
   ```

### 配置 Kafka 客户端

要在客户端配置SASL认证：

1. 在 producer.properties 或 consumer.properties 中为每个客户端配置 JAAS 配置属性。登录模块描述了生产者和消费者等客户端如何连接到 Kafka Broker。下面是一个为 PLAIN 机制的客户端配置的例子。

   ```properties
   sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
       username="alice" \
       password="alice-secret";
   ```

   选项 `username` 和 `password` 被客户端用来配置客户端连接的用户。在这个例子中，客户端以用户 *alice* 的身份连接到 broker。JVM中的不同客户端可以通过在 `sasl.jaas.config` 中指定不同的用户名和密码，以不同的用户身份连接。

   客户端的 JAAS 配置也可以指定为 JVM 参数，类似于[这里](https://kafka.apache.org/documentation/#security_client_staticjaas)描述的brokers。客户端使用名为 "KafkaClient "的登录部分。这个选项只允许一个JVM的所有客户端连接使用一个用户。

2. 在 producer.properties 或 consumer.properties 中配置以下属性:

   ```properties
   security.protocol=SASL_SSL
   sasl.mechanism=PLAIN
   ```

3. 在生产中使用 SASL/PLAIN

   - SASL/PLAIN 应该只与 SSL 作为传输层一起使用，以确保在没有加密的情况下不会在网上传输明确的密码。
   - Kafka中 SASL/PLAIN 的默认实现在 JAAS 配置文件中指定了用户名和密码，如[这里](https://kafka.apache.org/documentation/#security_sasl_plain_brokerconfig)所示。从Kafka 2.0版本开始，你可以通过配置自己的回调处理程序来避免在磁盘上存储清晰的密码，这些处理程序使用配置选项从外部来源获得用户名和密码 `sasl.server.callback.handler.class` and `sasl.client.callback.handler.class`.
   - 在生产系统中，外部认证服务器可以实现密码验证。从Kafka 2.0版本开始，你可以通过配置 `sasl.server.callback.handler.class` 插入你自己的回调处理程序，使用外部认证服务器进行密码验证。

## 5. 使用 SASL/SCRAM 进行认证

Salted Challenge Response Authentication Mechanism / SCRAM 是SASL机制的一个系列，解决了传统机制的安全问题，这些机制执行用户名/密码认证，如 PLAIN 和 DIGEST-MD5。该机制在 [RFC 5802](https://tools.ietf.org/html/rfc5802) 中定义。Kafka支持 [SCRAM-SHA-256](https://tools.ietf.org/html/rfc7677) 和 SCRAM-SHA-512，可以与TLS 一起使用，以执行安全认证。在 `principal.builder.class` 的默认实现下，用户名被用作认证的 `Principal`，用于配置ACL等。Kafka 的默认 SCRAM 实现在 Zookeeper 中存储 SCRAM 凭证，适合用于 Zookeeper 在私有网络上的 Kafka 安装。更多细节请参考[安全考虑](https://kafka.apache.org/documentation/#security_sasl_scram_security)。

### 创建 Creating SCRAM 证书

Kafka 中的SCRAM 实现使用 Zookeeper 作为凭证存储。可以使用 `kafka-configs.sh` 在 Zookeeper 中创建凭证。对于每个启用的SCRAM机制，必须通过添加机制名称的配置来创建凭证。在Kafka broker 启动之前，必须创建 broker 之间通信的凭证。客户端凭证可以动态地创建和更新，更新的凭证将用于验证新的连接。

为用户 *alice *创建 SCRAM 凭证，密码为 *alice-secret*：

```bash
> bin/kafka-configs.sh --zookeeper localhost:2182 --zk-tls-config-file zk_tls_config.properties --alter --add-config 'SCRAM-SHA-256=[iterations=8192,password=alice-secret],SCRAM-SHA-512=[password=alice-secret]' --entity-type users --entity-name alice
```

如果没有指定迭代次数，则使用默认的 4096 迭代次数。一个随机 salt 被创建，由salt、iterations、StoredKey 和 ServerKey 组成的SCRAM 身份被存储在 Zookeeper中。关于 SCRAM 身份和各个字段的详细信息，请参见[RFC 5802](https://tools.ietf.org/html/rfc5802)。

下面的例子还需要一个用户 *admin*，用于 broker 之间的通信，可以用以下方式创建。

```bash
> bin/kafka-configs.sh --zookeeper localhost:2182 --zk-tls-config-file zk_tls_config.properties --alter --add-config 'SCRAM-SHA-256=[password=admin-secret],SCRAM-SHA-512=[password=admin-secret]' --entity-type users --entity-name admin
```

现有的证书可以用 `--describe` 选项列出：

```bash
> bin/kafka-configs.sh --zookeeper localhost:2182 --zk-tls-config-file zk_tls_config.properties --describe --entity-type users --entity-name alice
```

可以使用 `--alter --delete-config` 选项删除一个或多个 SCRAM 机制的凭证： 

```bash
> bin/kafka-configs.sh --zookeeper localhost:2182 --zk-tls-config-file zk_tls_config.properties --alter --delete-config 'SCRAM-SHA-512' --entity-type users --entity-name alice
```

### 配置 Kafka Brokers

1. 在每个 Kafka broker 的配置目录下添加一个与下面类似的经过适当修改的 JAAS 文件，本例中我们称之为 kafka_server_jaas.conf:

   ```properties
   KafkaServer {
       org.apache.kafka.common.security.scram.ScramLoginModule required
       username="admin"
       password="admin-secret";
   };
   ```

   KafkaServer部分的属性 `username `和 `password `被 broker 用来启动与其他 broker 的连接。在这个例子中，`admin`是用于 broker 之间通信的用户。

2. Pass the JAAS config file location as JVM parameter to each Kafka broker:

   ```properties
   -Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf
   ```

3. 在 server.properties 中配置 SASL 端口和 SASL 机制，如上所述。比如:

   ```properties
   listeners=SASL_SSL://host.name:port
   security.inter.broker.protocol=SASL_SSL
   sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256 (or SCRAM-SHA-512)
   sasl.enabled.mechanisms=SCRAM-SHA-256 (or SCRAM-SHA-512)
   ```

### 配置 Kafka Clients

要在客户端配置SASL认证：

1. 在 producer.properties 或 consumer.properties 中为每个客户端配置JAAS配置属性。登录模块描述了生产者和消费者等客户端如何连接到 Kafka Broker。下面是一个为 SCRAM 机制配置客户端的例子。

   ```properties
   sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
       username="alice" \
       password="alice-secret";
   ```

   选项`username`和`password`被客户端用来配置客户端连接的用户。在这个例子中，客户端以用户*alice*的身份连接到 broker。JVM中的不同客户端可以通过在`sasl.jaas.config`中指定不同的用户名和密码，以不同的用户身份连接。

   客户端的 JAAS 配置也可以指定为 JVM 参数，类似于[这里](https://kafka.apache.org/documentation/#security_client_staticjaas)描述的brokers。客户端使用名为 "KafkaClient" 的登录部分。这个选项只允许一个JVM的所有客户端连接使用一个用户。

2. 在 producer.properties 或 consumer.properties 中配置以下属性:

   ```properties
   security.protocol=SASL_SSL
   sasl.mechanism=SCRAM-SHA-256 (or SCRAM-SHA-512)
   ```

### SASL/SCRAM 的安全考虑因素

- Kafka 中 SASL/SCRAM 的默认实现在 Zookeeper 中存储 SCRAM 凭证。这适合在 Zookeeper 是安全的并且在私有网络上的情况下生产使用。
- Kafka 只支持强散列函数 SHA-256 和 SHA-512，最小迭代次数为4096。如果 Zookeeper 的安全性受到影响，强散列函数与强密码和高迭代次数相结合，可以防止暴力攻击。
- SCRAM 应该只与 TLS 加密一起使用，以防止截获 SCRAM 的交换。这可以防止字典或暴力攻击，以及在 Zookeeper 被破坏的情况下防止冒充。
- 从 Kafka 2.0 版本开始，在 Zookeeper 不安全的情况下，可以通过配置 `sasl.server.callback.handler.class`，使用自定义回调处理程序覆盖默认的 SASL/SCRAM 凭证存储。
- 关于安全考虑的更多细节，请参考[RFC 5802](https://tools.ietf.org/html/rfc5802#section-9)。



## 6. 使用 SASL/OAUTHBEARER 进行认证

[OAuth 2授权框架](https://tools.ietf.org/html/rfc6749) "使第三方应用程序能够获得对HTTP服务的有限访问，可以通过协调资源所有者和HTTP服务之间的批准互动来代表资源所有者，或者允许第三方应用程序以自己的名义获得访问。" SASL OAUTHBEARER 机制能够在 SASL（即非HTTP）上下文中使用该框架；它在 [RFC 7628](https://tools.ietf.org/html/rfc7628) 中定义。Kafka 中默认的 OAUTHBEARER 实现创建和验证[不安全的JSON Web令牌](https://tools.ietf.org/html/rfc7515#appendix-A.5)，只适合在非生产性Kafka安装中使用。更多细节请参考[安全注意事项](https://kafka.apache.org/documentation/#security_sasl_oauthbearer_security)。

在 `principal.builder.class` 的默认实现下，OAuthBearerToken 的 principalName 被用作配置ACL等的认证 `Principal`。

### 配置 Kafka Brokers

1. 在每个Kafka broker 的配置目录下添加一个与下面类似的经过适当修改的JAAS文件，本例中我们称之为 kafka_server_jaas.conf：

   ```properties
   KafkaServer {
       org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required
       unsecuredLoginStringClaim_sub="admin";
   };
   ```

   `KafkaServer `部分的属性 `unsecuredLoginStringClaim_sub `是由经纪人在启动与其他经纪人的连接时使用。在这个例子中，`admin` 将出现在 subject (`sub`) claim 中，并将成为 broker 之间通信的用户。

2. 将 JAAS 配置文件的位置作为 JVM 参数传递给每个 Kafka broker:

   ```bash
   -Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf
   ```

3. 在 server.properties 中配置 SASL 端口和 SAS L机制，如上所述。比如:

   ```properties
   listeners=SASL_SSL://host.name:port (or SASL_PLAINTEXT if non-production)
   security.inter.broker.protocol=SASL_SSL (or SASL_PLAINTEXT if non-production)
   sasl.mechanism.inter.broker.protocol=OAUTHBEARER
   sasl.enabled.mechanisms=OAUTHBEARER
   ```

### 配置 Kafka Clients

要在客户端配置SASL认证:

1. 在producer.properties或consumer.properties中为每个客户端配置JAAS配置属性。登录模块描述了生产者和消费者等客户端如何连接到Kafka Broker。下面是一个OAUTHBEARER机制的客户端的配置例子:

   ```properties
   sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
       unsecuredLoginStringClaim_sub="alice";
   ```

   选项 `unsecuredLoginStringClaim_sub` 被客户用来配置 subject (`sub`) claim，它决定了客户端连接的用户。在这个例子中，客户端以用户 *alice* 的身份连接到 broker 。一个JVM中的不同客户可以通过在`sasl.jaas.config`中指定不同的 subject (`sub`) claim，以不同的用户身份连接。

   客户端的JAAS配置也可以指定为JVM参数，类似于[这里](https://kafka.apache.org/documentation/#security_client_staticjaas)描述的 brokers。客户端使用名为 "KafkaClient" 的登录部分。这个选项只允许一个JVM的所有客户端连接使用一个用户。

2. 在 producer.properties 或 consumer.properties 中配置以下属性：

   ```properties
   security.protocol=SASL_SSL (or SASL_PLAINTEXT if non-production)
   sasl.mechanism=OAUTHBEARER
   ```

3. SASL/OAUTHBEARER 的默认实现依赖于 jackson-databind 库。由于它是一个可选的依赖关系，用户必须通过他们的构建工具将其配置为依赖关系。

### SASL/OAUTHBEARER 的无担保令牌创建选项

- Kafka 中 SASL/OAUTHBEARER 的默认实现可以创建和验证不安全的 JSON Web 令牌。虽然只适合于非生产性使用，但它确实提供了在DEV或TEST环境下创建任意令牌的灵活性。

- 下面是客户端支持的各种 JAAS 模块选项（如果 OAUTHBEARER 是 broker 之间的协议，则在 broker 一侧也支持）。

  | 创建无担保令牌的 JAAS 模块选项                    | 文档                                                         |
  | :------------------------------------------------ | :----------------------------------------------------------- |
  | `unsecuredLoginStringClaim_<claimname>="value"`   | 创建一个具有给定名称和值的`String` claim。除了'iat'和'exp'（这些是自动生成的），任何有效的 claim 名称都可以被指定。 |
  | `unsecuredLoginNumberClaim_<claimname>="value"`   | 创建一个具有给定名称和值的 `Number` claim。除了'iat'和'exp'（这些是自动生成的），任何有效的 claim 名称都可以被指定。 |
  | `unsecuredLoginListClaim_<claimname>="value"`     | 创建一个 `String List` claim ，具有给定的名称和从给定值中解析出来的值，其中第一个字符被当作分隔符。例如： `unsecuredLoginListClaim_fubar="|value1|value2"`。除了'iat'和'exp'（这些是自动生成的），任何有效的 claim 名称都可以被指定。 |
  | `unsecuredLoginExtension_<extensionname>="value"` | 创建一个具有给定名称和值的`String`扩展。例如：`unsecuredLoginExtension_traceId="123"`。一个有效的扩展名是任何小写或大写的字母序列。此外，"auth "扩展名被保留。一个有效的扩展值是ASCII码1-127的任何字符组合。 |
  | `unsecuredLoginPrincipalClaimName`                | 如果你希望持有 principal 名称的 "String" claim 要求的名称不是 "sub"，则设置为自定义 claim 要求名称。 |
  | `unsecuredLoginLifetimeSeconds`                   | 如果要将令牌过期时间设置为默认值3600秒（即1小时）之外的其他值，则设置为一个整数。`exp`  claim 将被设置为反映过期时间。 |
  | `unsecuredLoginScopeClaimName`                    | 如果你希望持有任何标记范围的 "String" 或 "String List" claim 的名称不是 "scope"，则设置为自定义 claim 名称。 |

### SASL/OAUTHBEARER 的无担保令牌验证选项

- 以下是经纪人方面支持的各种JAAS模块选项，用于无担保的JSON网络令牌验证：

  | JAAS Module Option for Unsecured Token Validation | Documentation                                                |
  | :------------------------------------------------ | :----------------------------------------------------------- |
  | `unsecuredValidatorPrincipalClaimName="value"`    | Set to a non-empty value if you wish a particular `String` claim holding a principal name to be checked for existence; the default is to check for the existence of the '`sub`' claim. |
  | `unsecuredValidatorScopeClaimName="value"`        | Set to a custom claim name if you wish the name of the `String` or `String List` claim holding any token scope to be something other than '`scope`'. |
  | `unsecuredValidatorRequiredScope="value"`         | Set to a space-delimited list of scope values if you wish the `String/String List` claim holding the token scope to be checked to make sure it contains certain values. |
  | `unsecuredValidatorAllowableClockSkewMs="value"`  | Set to a positive integer value if you wish to allow up to some number of positive milliseconds of clock skew (the default is 0). |

- 默认的不安全的 SASL/OAUTHBEARER 实现可以使用自定义的登录和SASL服务器回调处理程序进行重写（在生产环境中必须重写）。

- 关于安全考虑的更多细节，请参考[RFC 6749, Section 10](https://tools.ietf.org/html/rfc6749#section-10)。

### 为 SASL/OAUTHBEARER 刷新令牌

Kafka 会在任何令牌过期前定期刷新，这样客户端就可以继续与 broker 建立连接。影响刷新算法操作方式的参数被指定为生产者/消费者/ broker 配置的一部分，具体如下。详细情况见其他地方的这些属性的文档。默认值通常是合理的，在这种情况下，这些配置参数就不需要明确设置。

| Producer/Consumer/Broker 配置属性       |
| :-------------------------------------- |
| `sasl.login.refresh.window.factor`      |
| `sasl.login.refresh.window.jitter`      |
| `sasl.login.refresh.min.period.seconds` |
| `sasl.login.refresh.min.buffer.seconds` |

### SASL/OAUTHBEARER 的安全/生产使用

生产用例将需要编写 `org.apache.kafka.common.security.auth.AuthenticateCallbackHandler` 的实现，该实现可以处理`org.apache.kafka.common.security.oauthbearer.OAuthBearerTokenCallback` 的实例，并通过 `sasl.login.callback.handler.class` 配置选项来处理非 broker 客户端，或者通过 `Listener.name.ssl.oauthbearer.sasl.login.callback.handler.class` 配置选项来处理 broker（当 `SASL/OAUTHBEARER` 为 broker 间协议时）。

生产用例还需要编写 `org.apache.kafka.common.security.auth.AuthenticateCallbackHandler` 的实现，该实现可以处理`org.apache.kafka.common.security.oauthBearerValidatorCallback` 的实例，并通过 `listener.name.sasl_ssl.oauthbearer.sasl.server.callback.handler.class broker` 配置选项声明它。

### SASL/OAUTHBEARER 的安全考虑因素

- Kafka 中 SASL/OAUTHBEARER 的默认实现创建并验证[无担保的JSON Web令牌](https://tools.ietf.org/html/rfc7515#appendix-A.5)。这只适合于非生产使用。
- 在生产环境中，OAUTHBEARER 应该只使用 TLS 加密，以防止截获令牌。
- 默认的不安全的 SASL/OAUTHBEARER 实现可以使用上述的自定义登录和SASL服务器回调处理程序进行覆盖（在生产环境中必须进行覆盖）。
- 关于OAuth 2一般安全考虑的更多细节，请参考[RFC 6749，第10节](https://tools.ietf.org/html/rfc6749#section-10)。

## 7. 在 broker 中启用多个SASL机制

1. 在 JAAS 配置文件的 KafkaServer 部分指定所有启用机制的登录模块的配置。比如说:

   ```properties
   KafkaServer {
       com.sun.security.auth.module.Krb5LoginModule required
       useKeyTab=true
       storeKey=true
       keyTab="/etc/security/keytabs/kafka_server.keytab"
       principal="kafka/kafka1.hostname.com@EXAMPLE.COM";
   
       org.apache.kafka.common.security.plain.PlainLoginModule required
       username="admin"
       password="admin-secret"
       user_admin="admin-secret"
       user_alice="alice-secret";
   };
   ```

2. 在 server.properties 中启用 SASL 机制:

   ```properties
   sasl.enabled.mechanisms=GSSAPI,PLAIN,SCRAM-SHA-256,SCRAM-SHA-512,OAUTHBEARER
   ```

3. 如果需要，在 server.properties 中指定 SASL 安全协议和 broker 之间的通信机制：

   ```properties
   security.inter.broker.protocol=SASL_PLAINTEXT (or SASL_SSL)
   sasl.mechanism.inter.broker.protocol=GSSAPI (or one of the other enabled mechanisms)
   ```

4. 按照 [GSSAPI (Kerberos)](https://kafka.apache.org/documentation/#security_sasl_kerberos_brokerconfig)、[PLAIN](https://kafka.apache.org/documentation/#security_sasl_plain_brokerconfig)、[SCRAM ](https://kafka.apache.org/documentation/#security_sasl_scram_brokerconfig)和 [OAUTHBEARER](https://kafka.apache.org/documentation/#security_sasl_oauthbearer_brokerconfig) 中特定机制的步骤，为启用的机制配置SASL。

## 8. 在运行中的集群中修改 SASL 机制

在运行中的集群中，可以用以下顺序修改SASL机制:

1. 启用新的 SASL 机制，将该机制添加到每个代理的 server.properties 中的 `sasl.enabled. mechanisms`。按照[这里](https://kafka.apache.org/documentation/#security_sasl_multimechanism)的描述，更新JAAS配置文件以包括两种机制。逐步跳转集群节点。
2. 使用新机制重新启动客户。
3. 要改变 broker 之间的通信机制（如果需要的话），将 server.properties 中的`sasl.mechanism.inter.broker.protocol`设置为新的机制，并再次递增地弹出集群。
4. 要删除旧机制（如果需要的话），从 server.properties 的 `sasl.enabled. mechanisms `中删除旧机制，并从JAAS配置文件中删除旧机制的条目。再次递增弹出集群。

## 9. 使用授权令牌进行认证

基于委托令牌的认证是一种轻量级的认证机制，以补充现有的 SASL/SSL 方法。委托令牌是 kafka broker 和客户端之间的共享秘密。委托令牌将帮助处理框架在安全环境下将工作负载分配给可用的工作者，而不需要在使用2-way SSL时分配Kerberos TGT/keytabs或密钥存储的额外成本。参见[KIP-48](https://cwiki.apache.org/confluence/display/KAFKA/KIP-48+Delegation+token+support+for+Kafka)了解更多细节。

在 principal.builder.class 的默认实现下，授权令牌的所有者被用作配置 ACL 等的认证 Principal。

使用授权令牌的典型步骤是:

1. 用户通过SASL或SSL与Kafka集群进行认证，并获得一个授权令牌。这可以通过Admin APIs或使用`kafka-delegation-tokens.sh`脚本完成。
2. 用户安全地将授权令牌传递给Kafka客户端，以便与Kafka集群进行认证。
3. 代币所有者/更新者可以更新/终止委托代币。

### Token Management

A secret is used to generate and verify delegation tokens. This is supplied using config option `delegation.token.secret.key`. The same secret key must be configured across all the brokers. If the secret is not set or set to empty string, brokers will disable the delegation token authentication.

In the current implementation, token details are stored in Zookeeper and is suitable for use in Kafka installations where Zookeeper is on a private network. Also currently, this secret is stored as plain text in the server.properties config file. We intend to make these configurable in a future Kafka release.

A token has a current life, and a maximum renewable life. By default, tokens must be renewed once every 24 hours for up to 7 days. These can be configured using `delegation.token.expiry.time.ms` and `delegation.token.max.lifetime.ms` config options.

Tokens can also be cancelled explicitly. If a token is not renewed by the token’s expiration time or if token is beyond the max life time, it will be deleted from all broker caches as well as from zookeeper.

### Creating Delegation Tokens

Tokens can be created by using Admin APIs or using `kafka-delegation-tokens.sh` script. Delegation token requests (create/renew/expire/describe) should be issued only on SASL or SSL authenticated channels. Tokens can not be requests if the initial authentication is done through delegation token. A token can be created by the user for that user or others as well by specifying the `--owner-principal` parameter. Owner/Renewers can renew or expire tokens. Owner/renewers can always describe their own tokens. To describe other tokens, a DESCRIBE_TOKEN permission needs to be added on the User resource representing the owner of the token. `kafka-delegation-tokens.sh` script examples are given below.

Create a delegation token:

```bash
> bin/kafka-delegation-tokens.sh --bootstrap-server localhost:9092 --create   --max-life-time-period -1 --command-config client.properties --renewer-principal User:user1
```

Create a delegation token for a different owner:

```bash
> bin/kafka-delegation-tokens.sh --bootstrap-server localhost:9092 --create   --max-life-time-period -1 --command-config client.properties --renewer-principal User:user1 --owner-principal User:owner1
```

Renew a delegation token:

```bash
> bin/kafka-delegation-tokens.sh --bootstrap-server localhost:9092 --renew    --renew-time-period -1 --command-config client.properties --hmac ABCDEFGHIJK
```

Expire a delegation token:

```bash
> bin/kafka-delegation-tokens.sh --bootstrap-server localhost:9092 --expire   --expiry-time-period -1   --command-config client.properties  --hmac ABCDEFGHIJK
```

Existing tokens can be described using the --describe option:

```bash
> bin/kafka-delegation-tokens.sh --bootstrap-server localhost:9092 --describe --command-config client.properties  --owner-principal User:user1
```

### Token Authentication

Delegation token authentication piggybacks on the current SASL/SCRAM authentication mechanism. We must enable SASL/SCRAM mechanism on Kafka cluster as described in [here](https://kafka.apache.org/documentation/#security_sasl_scram).

Configuring Kafka Clients:

1. Configure the JAAS configuration property for each client in producer.properties or consumer.properties. The login module describes how the clients like producer and consumer can connect to the Kafka Broker. The following is an example configuration for a client for the token authentication:

   ```text
   sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
       username="tokenID123" \
       password="lAYYSFmLs4bTjf+lTZ1LCHR/ZZFNA==" \
       tokenauth="true";
   ```

   The options `username` and `password` are used by clients to configure the token id and token HMAC. And the option `tokenauth` is used to indicate the server about token authentication. In this example, clients connect to the broker using token id: *tokenID123*. Different clients within a JVM may connect using different tokens by specifying different token details in `sasl.jaas.config`.

   JAAS configuration for clients may alternatively be specified as a JVM parameter similar to brokers as described [here](https://kafka.apache.org/documentation/#security_client_staticjaas). Clients use the login section named `KafkaClient`. This option allows only one user for all client connections from a JVM.

### Procedure to manually rotate the secret

We require a re-deployment when the secret needs to be rotated. During this process, already connected clients will continue to work. But any new connection requests and renew/expire requests with old tokens can fail. Steps are given below.

1. Expire all existing tokens.
2. Rotate the secret by rolling upgrade, and
3. Generate new tokens

We intend to automate this in a future Kafka release.

