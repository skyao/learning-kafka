# 安装

## 下载

从 kafka 官网的下载页面 [http://kafka.apache.org/downloads](http://kafka.apache.org/downloads) 下载最新版本。

## 安装

解压缩下载下来的文件(如 kafka_2.12-0.10.2.0.tgz)到合适的位置，然后修改 `/etc/profile`,加入下列内容：

```bash
# kafka
export KAFKA_HOME=/home/sky/work/soft/kafka
export PATH=$PATH:$KAFKA_HOME/bin
```

通过 `source /etc/profile` 命令载入。


## 最简启动

以最简单的方式启动 kafka 并启动 producer 和 consumer：

```bash
cd /home/sky/work/soft/kafka

# start zookeeper
zookeeper-server-start.sh config/zookeeper.properties

# start kafka server
kafka-server-start.sh config/server.properties

# create topic

kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic topic1

# Starting producer
kafka-console-producer.sh --broker-list localhost:9092 --topic topic1

# start comsumer
kafka-console-consumer.sh --zookeeper localhost:2181 --topic topic1 --from-beginning
```

之后在 producer 的控制台中输入内容，回车，就可以在 consumer 的控制台中看到对应的内容。




