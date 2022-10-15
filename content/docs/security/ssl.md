---
title: "SSL"
linkTitle: "SSL"
weight: 30
date: 2022-06-27
description: >
  使用SSL进行加密和认证
---

> https://kafka.apache.org/documentation/#security_ssl

Apache Kafka 允许客户端使用 SSL 对流量进行加密以及验证。默认情况下，SSL 是禁用的，但如果需要，可以打开。下面几段详细解释了如何设置你自己的 PKI 基础设施，用它来创建证书并配置 Kafka 来使用这些证书。



## 1. 为每个Kafka broker 生成SSL密钥和证书

部署一个或多个支持 SSL 的 broker 的第一步是为每个服务器生成一个公钥/私钥对。由于 Kafka 希望所有的密钥和证书都存储在密钥库中，我们将使用 Java 的 keytool 命令来完成这项任务。该工具支持两种不同的钥匙库格式，一种是 Java 特有的 jks 格式，现在已经废弃了，另一种是 PKCS12。PKCS12 是 Java 9 版本的默认格式，为了确保这个格式被使用，无论使用的是什么 Java 版本，以下所有命令都明确指定了 PKCS12 格式。

```bash
keytool -keystore {keystorefile} -alias localhost -validity {validity} -genkey -keyalg RSA -storetype pkcs12
```

你需要在上述命令中指定两个参数：

1. keystorefile: 存储该 broker 的密钥（以及后来的证书）的keystore文件。keystore文件包含了这个 broker 的私钥和公钥，因此它需要被妥善保存。理想情况下，这一步是在密钥将被用于的 Kafka broker 上运行，因为这个密钥不应该被传输/离开它所要使用的服务器。
2. validity: 密钥的有效时间，以天为单位。请注意，这与证书的有效期不同，后者将在 [签署证书](https://kafka.apache.org/documentation/#security_ssl_signing) 中确定。你可以用同一把密钥申请多个证书：如果你的密钥有效期为10年，但你的CA只签署有效期为一年的证书，你可以在一段时间内用同一把密钥签署10个证书。

为了获得可与刚刚创建的私钥一起使用的证书，需要创建一个证书签署请求。该签名请求由受信任的 CA 签署后会产生实际的证书，然后可以安装在钥匙库中并用于验证目的。

为了生成证书签署请求，对迄今为止创建的所有服务器密钥库运行以下命令：

```bash
keytool -keystore server.keystore.jks -alias localhost -validity {validity} -genkey -keyalg RSA -destkeystoretype pkcs12 -ext SAN=DNS:{FQDN},IP:{IPADDRESS1}
```

该命令假定你想在证书中添加主机名信息，如果不是这样，你可以省略扩展参数 `-ext SAN=DNS:{FQDN},IP:{IPADDRESS1}`。请参阅下文以了解更多相关信息。

### 主机名称验证

启用主机名验证时，是将你正在连接的服务器所出示的证书的属性与该服务器的实际主机名或IP地址进行核对的过程，以确保你确实正在连接到正确的服务器。

这种检查的主要原因是为了防止中间人攻击。对于Kafka来说，这种检查在很长一段时间内都是默认禁用的，但从Kafka 2.0.0开始，服务器的主机名验证在客户端连接和代理间连接中都是默认启用的。

可以通过设置 `ssl.endpoint.identification.algorithm` 为空字符串来禁用服务器主机名验证。

对于动态配置的 broker 监听器，可以使用 `kafka-configs.sh` 禁用主机名验证：

```bash
> bin/kafka-configs.sh --bootstrap-server localhost:9093 --entity-type brokers --entity-name 0 --alter --add-config "listener.name.internal.ssl.endpoint.identification.algorithm="
```

**注意:**

通常情况下，没有任何好的理由禁用主机名验证，除非是 "只是让它工作" 的最快方法，并承诺 "以后有更多时间时再修复它"!

在正确的时间进行主机名验证并不难，但一旦集群开始运行，就会变得更加困难--帮你自己一个忙，现在就去做吧！。

如果启用了主机名验证，客户将根据以下两个字段之一验证服务器的完全合格域名（FQDN）或ip地址。

1. Common Name (CN)
2. [Subject Alternative Name (SAN)](https://tools.ietf.org/html/rfc5280#section-4.2.1.6)

虽然 Kafka 检查这两个字段，但自2000年以来，使用通用名称（common name/CN）字段进行主机名验证已经[废弃](https://tools.ietf.org/html/rfc2818#section-3.1)，应尽可能避免使用。此外，SAN字段更加灵活，允许在一个证书中声明多个 DNS 和 IP 条目。

另一个好处是，如果 SAN 字段被用于主机名验证，为了授权目的，可以将通用名称设置为一个更有意义的值。由于我们需要 SAN 字段包含在已签署的证书中，它将在生成签署请求时被指定。它也可以在生成密钥对时指定，但这不会自动被复制到签名请求中。

要添加一个SAN字段，请在 keytool 命令中添加以下参数 `-ext SAN=DNS:{FQDN},IP:{IPADDRESS}`。

 ## 2. 创建自己的CA

在这一步之后，集群中的每台机器都有一个公钥/私钥对，已经可以用来加密流量，还有一个证书签署请求，这是创建证书的基础。为了增加认证功能，这个签名请求需要由一个受信任的机构签署，这个机构将在这一步中创建。

证书颁发机构（CA）负责签署证书。CA的工作就像一个签发护照的政府--政府在每本护照上盖章（签名），这样护照就很难伪造了。其他政府会验证这些印章以确保护照的真实性。同样，CA签署证书，密码学保证签署的证书在计算上很难被伪造。因此，只要CA是一个真实可信的机构，客户就有强大的保证，他们正在连接到真实的机器。

在本指南中，我们将成为我们自己的证书颁发机构。当在企业环境中建立一个生产集群时，这些证书通常会由整个公司信任的企业CA签署。请参阅[生产中的常见陷阱](https://kafka.apache.org/documentation/#security_ssl_production)了解这种情况下需要考虑的一些问题。

由于OpenSSL的一个[bug]()，x509模块不会将CSR中要求的扩展字段复制到最终证书中。由于我们希望SAN扩展在我们的证书中存在，以实现主机名验证，我们将使用*ca*模块来代替。这需要在我们生成CA密钥对之前进行一些额外的配置。

将以下列表保存到一个名为 openssl-ca.cnf 的文件中，并根据需要调整有效性和共同属性的值：

```properties
HOME            = .
RANDFILE        = $ENV::HOME/.rnd

####################################################################
[ ca ]
default_ca    = CA_default      # The default ca section

[ CA_default ]

base_dir      = .
certificate   = $base_dir/cacert.pem   # The CA certifcate
private_key   = $base_dir/cakey.pem    # The CA private key
new_certs_dir = $base_dir              # Location for new certs after signing
database      = $base_dir/index.txt    # Database index file
serial        = $base_dir/serial.txt   # The current serial number

default_days     = 1000         # How long to certify for
default_crl_days = 30           # How long before next CRL
default_md       = sha256       # Use public key default MD
preserve         = no           # Keep passed DN ordering

x509_extensions = ca_extensions # The extensions to add to the cert

email_in_dn     = no            # Don't concat the email in the DN
copy_extensions = copy          # Required to copy SANs from CSR to cert

####################################################################
[ req ]
default_bits       = 4096
default_keyfile    = cakey.pem
distinguished_name = ca_distinguished_name
x509_extensions    = ca_extensions
string_mask        = utf8only

####################################################################
[ ca_distinguished_name ]
countryName         = Country Name (2 letter code)
countryName_default = DE

stateOrProvinceName         = State or Province Name (full name)
stateOrProvinceName_default = Test Province

localityName                = Locality Name (eg, city)
localityName_default        = Test Town

organizationName            = Organization Name (eg, company)
organizationName_default    = Test Company

organizationalUnitName         = Organizational Unit (eg, division)
organizationalUnitName_default = Test Unit

commonName         = Common Name (e.g. server FQDN or YOUR name)
commonName_default = Test Name

emailAddress         = Email Address
emailAddress_default = test@test.com

####################################################################
[ ca_extensions ]

subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always, issuer
basicConstraints       = critical, CA:true
keyUsage               = keyCertSign, cRLSign

####################################################################
[ signing_policy ]
countryName            = optional
stateOrProvinceName    = optional
localityName           = optional
organizationName       = optional
organizationalUnitName = optional
commonName             = supplied
emailAddress           = optional

####################################################################
[ signing_req ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer
basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment
```

然后创建一个数据库和序列号文件，这些将被用来跟踪哪些证书是用这个CA签署的。这两个文件都是简单的文本文件，与你的CA密钥存放在同一个目录下。

```bash
> echo 01 > serial.txt
> touch index.txt
```

完成这些步骤后，你现在就可以生成你的CA了，以后将用于签署证书。

```bash
> openssl req -x509 -config openssl-ca.cnf -newkey rsa:4096 -sha256 -nodes -out cacert.pem -outform PEM
```

CA只是一个公共/私人密钥对和证书，它是由自己签署的，而且只用于签署其他证书。

这个密钥对应该非常安全，如果有人获得了这个密钥对，他们可以创建和签署将被你的基础设施所信任的证书，这意味着他们在连接到任何信任这个CA的服务时可以冒充任何人。

下一步是将生成的CA添加到**客户的信任库(truststore)**，这样客户就可以信任这个CA。

```bash
> keytool -keystore client.truststore.jks -alias CARoot -import -file ca-cert
```

**Note:** If you configure the Kafka brokers to require client authentication by setting ssl.client.auth to be "requested" or "required" in the [Kafka brokers config](https://kafka.apache.org/documentation/#brokerconfigs) then you must provide a truststore for the Kafka brokers as well and it should have all the CA certificates that clients' keys were signed by.

**注意：**如果你通过在 [Kafka brokers config](https://kafka.apache.org/documentation/#brokerconfigs) 中设置 ssl.client.auth 为 "request "或 "required"，将 Kafka brokers 配置为需要客户端认证，那么你也必须为 Kafka brokers 提供一个信任仓库，它应该有所有客户端的密钥由CA证书签署。

```bash
> keytool -keystore server.truststore.jks -alias CARoot -import -file ca-cert
```

与步骤1中存储每台机器自身身份的钥匙库不同，客户的信任库(truststore)存储了客户应该信任的所有证书。将一个证书导入自己的信任库（truststore）也意味着信任由该证书签署的所有证书。正如上面的类比，信任政府（CA）也意味着信任它所签发的所有护照（证书）。这个属性被称为信任链（chain of trust），在大型 Kafka 集群上部署 SSL 时，它特别有用。你可以用 CA 来签署集群中的所有证书，并让所有机器共享同一个信任 CA 的信任库。这样一来，所有的机器都可以验证其他所有的机器。



## 3. 签署证书

然后与 CA 一起签署：

```bash
> openssl ca -config openssl-ca.cnf -policy signing_policy -extensions signing_req -out {server certificate} -infiles {certificate signing request}
```

最后，你需要把CA的证书和签名的证书都导入 keystore：

```bash
> keytool -keystore {keystore} -alias CARoot -import -file {CA certificate}
> keytool -keystore {keystore} -alias localhost -import -file cert-signed
```

参数的定义如下：

1. keystore: keystore 的位置
2. CA certificate: CA的证书
3. certificate signing request: 用服务器密钥创建的csr
4. server certificate: 将服务器的签名证书写入该文件。

这将给你留下一个名为 *truststore.jks* 的 truststore -- 这对所有客户端和 broker 来说都是一样的，不包含任何敏感信息，所以没有必要保护它。

此外，你将在每个节点上有一个*server.keystore.jks*文件，其中包含该节点的密钥、证书和你的CAs证书，关于如何使用这些文件的信息，请参考 配置Kafka Brokers 和 配置Kafka Client。

关于这个话题的一些工具帮助，请查 [easyRSA 项目，它有大量的脚本来帮助完成这些步骤。

### PEM格式的SSL密钥和证书

从2.7.0开始，可以在配置中直接为 Kafka broker 和客户端配置 PEM 格式的 SSL 密钥和信任存储。这避免了在文件系统上存储单独的文件，并受益于 Kafka 配置的密码保护功能。除了 JKS 和 PKCS12 之外，PEM也可以作为基于文件的密钥和信任存储的存储类型。要在代理或客户端配置中直接配置PEM密钥存储，应在 `ssl.keystore.key` 中提供PEM格式的私钥，在 `ssl.keystore.certificate.chain` 中提供 PEM 格式的证书链。为了配置信任存储，应在 `ssl.truststore.certificates` 中提供信任证书，如CA的公共证书。由于 PEM 通常以多行 base-64 字符串的形式存储，配置值可以作为多行字符串包含在 Kafka 配置中，每行以反斜杠 ('\') 为结尾进行续行。

存储密码配置 `ssl.keystore.password` 和 `ssl.truststore.password` 不用于PEM。如果私钥是用密码加密的，必须在 `ssl.key.password` 中提供密钥密码。私钥可以以未加密的形式提供，无需密码。在生产部署中，在这种情况下，配置应该被加密或使用 Kafka 中的密码保护功能进行外部化。注意，当使用 OpenSSL 等外部工具进行加密时，默认的 SSL 引擎工厂对加密的私钥解密能力有限。像 BouncyCastle 这样的第三方库可以与自定义的`SslEngineFactory`集成，以支持更广泛的加密私钥。

## 4. 生产中的常见陷阱

上述段落展示了创建自己的CA并使用它为集群签署证书的过程。虽然对sandbox、dev、测试和类似的系统非常有用，但这通常不是为企业环境中的生产集群创建证书的正确过程。企业通常会操作他们自己的CA，用户可以发送 CSR 到这个 CA 上签名，这样做的好处是用户不需要负责保持 CA 的安全，同时也是一个大家可以信任的中央机构。然而，它也从用户那里拿走了对签署证书过程的很多控制权。很多时候，操作企业 CA 的人都会对证书施加严格的限制，当试图在 Kafka上 使用这些证书时，就会产生问题。

1. [扩展密钥的使用](https://tools.ietf.org/html/rfc5280#section-4.2.1.12)

   证书可能包含一个扩展字段，控制证书的使用目的。如果该字段为空，则对用途没有限制，但如果在这里指定了任何用途，有效的SSL实现必须执行这些用途。

   Kafka的相关用途是:

   - 客户端认证
   - 服务器认证

   Kafka broker 需要允许这两种用法，因为对于集群内的通信，每个 broker 都会对其他 broker 表现得既是客户端又是服务器。企业CA有一个Web服务器的签名配置文件，并将其用于Kafka，这并不罕见，它将只包含serverAuth使用值，并导致SSL握手失败。

2. **中级证书**
   为了安全起见，企业根CA经常保持离线状态。为了保证日常使用，我们创建了所谓的中间CA，然后用它们来签署最终的证书。当把由中间CA签署的证书导入钥匙库时，有必要提供整个信任链，直到根CA。这可以通过简单的*cat*将证书文件合并成一个证书文件，然后用keytool导入来完成。

3. 未能复制扩展字段

   CA运营商通常不愿意从CSR中复制和要求的扩展字段，而倾向于自己指定这些字段，因为这使得恶意的一方更难获得具有潜在的误导性或欺诈性价值的证书。建议仔细检查已签署的证书，这些证书是否包含所有要求的SAN字段，以实现正确的主机名验证。下面的命令可以用来将证书细节打印到控制台，并与最初要求的内容进行比较。

   ```bash
   > openssl x509 -in certificate.crt -text -noout
   ```

## 5. 配置 Kafka Brokers

如果没有为 broker 之间的通信启用SSL（如何启用见下文），那么PLAINTEXT和SSL端口都是必要的。

```properties
listeners=PLAINTEXT://host.name:port,SSL://host.name:port
```

在 broker 方面，需要进行以下SSL配置:

```properties
ssl.keystore.location=/var/private/ssl/server.keystore.jks
ssl.keystore.password=test1234
ssl.key.password=test1234
ssl.truststore.location=/var/private/ssl/server.truststore.jks
ssl.truststore.password=test1234
```

注意：`ssl.truststore.password` 在技术上是可选的，但强烈推荐。如果不设置密码，仍然可以访问 truststore，但完整性检查被禁用。值得考虑的可选设置:

1. ssl.client.auth=none ("required" => 需要客户认证, "requested" => 要求客户认证，没有证书的客户仍然可以连接。我们不鼓励使用 "requested"，因为它提供了一种错误的安全感，配置错误的客户仍然会成功连接。)
2. ssl.cipher.suites (Optional). 一个密码套件是认证、加密、MAC和密钥交换算法的命名组合，用于协商使用TLS或SSL网络协议的网络连接的安全设置。(默认是一个空列表)
3. ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1 (列出你要接受的客户的SSL协议。请注意，SSL已被弃用，取而代之的是TLS，不建议在生产中使用SSL)
4. ssl.keystore.type=JKS
5. ssl.truststore.type=JKS
6. ssl.secure.random.implementation=SHA1PRNG

如果你想为 broker 之间的通信启用SSL，请在 server.properties 文件中添加以下内容（它默认为PLAINTEXT）。

```text
security.inter.broker.protocol=SSL
```

由于一些国家的进口法规，Oracle的实现限制了默认情况下可用的加密算法的强度。如果需要更强的算法（例如，256位密钥的AES），必须在JDK/JRE中获得并安装JCE无限强度管辖政策文件。更多信息请参见JCA供应商文档。

JRE/JDK将有一个默认的伪随机数发生器（PRNG），用于加密操作，所以不需要配置与`ssl.secure.random.implement`一起使用的实现。然而，一些实现存在性能问题（特别是Linux系统上选择的默认实现`NativePRNG`，利用了全局锁）。在SSL连接的性能成为问题的情况下，考虑明确设置要使用的实现。`SHA1PRNG`的实现是无阻塞的，并在重载下显示出非常好的性能特征（50MB/秒的生产消息，加上复制流量，每个经纪人）。

一旦你启动 broker，你应该能够在server.log中看到:

```properties
with addresses: PLAINTEXT -> EndPoint(192.168.64.1,9092,PLAINTEXT),SSL -> EndPoint(192.168.64.1,9093,SSL)
```

要快速检查服务器钥匙库和信任库是否设置正确，你可以运行以下命令

```bash
> openssl s_client -debug -connect localhost:9093 -tls1
```

(注意：TLSv1应该列在ssl.enabled.protocols下)

在这个命令的输出中，你应该看到服务器的证书:

```text
-----BEGIN CERTIFICATE-----
{variable sized random bytes}
-----END CERTIFICATE-----
subject=/C=US/ST=CA/L=Santa Clara/O=org/OU=org/CN=Sriharsha Chintalapani
issuer=/C=US/ST=CA/L=Santa Clara/O=org/OU=org/CN=kafka/emailAddress=test@test.com
```

如果证书没有显示出来，或者有任何其他错误信息，那么你的钥匙库并没有正确设置。

## 6. 配置 Kafka Clients

只有新的 Kafka 生产者和消费者支持 SSL，旧的 API 不支持。对于生产者和消费者来说，SSL 的配置将是相同的。

如果 broker 中不需要客户端认证，那么下面是一个最小的配置例子：

```properties
security.protocol=SSL
ssl.truststore.location=/var/private/ssl/client.truststore.jks
ssl.truststore.password=test1234
```

注意：ssl.truststore.password 在技术上是可选的，但强烈推荐。如果不设置密码，仍然可以访问truststore，但完整性检查被禁用。如果需要客户端认证，那么必须像第1步那样创建一个密钥库，同时还必须配置以下内容。

```properties
ssl.keystore.location=/var/private/ssl/client.keystore.jks
ssl.keystore.password=test1234
ssl.key.password=test1234
```

根据我们的要求和 broker 的配置，可能还需要其他的配置设置:

1. ssl.provider (Optional). 用于SSL连接的安全提供者的名称。默认值是JVM的默认安全提供者。
2. ssl.cipher.suites (Optional). 密码套件是认证、加密、MAC和密钥交换算法的命名组合，用于协商使用TLS或SSL网络协议的网络连接的安全设置。
3. ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1. 它应该至少列出在 broker 一方配置的一个协议
4. ssl.truststore.type=JKS
5. ssl.keystore.type=JKS

使用控制台-生产者和控制台-消费者的例子:

```bash
> kafka-console-producer.sh --bootstrap-server localhost:9093 --topic test --producer.config client-ssl.properties
> kafka-console-consumer.sh --bootstrap-server localhost:9093 --topic test --consumer.config client-ssl.properties
```



