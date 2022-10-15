---
title: "在ubuntu上安装kafka"
linkTitle: "ubuntu"
weight: 10
date: 2022-06-27
description: >
  在ubuntu实体机器上安装Kafka
---



## 安装

### 下载

从 kafka 官网的下载页面 [http://kafka.apache.org/downloads](http://kafka.apache.org/downloads) 下载最新版本。

### 解压缩

解压缩下载下来的文件(如 kafka_2.13-3.3.1.tgz)到合适的位置，然后修改 `/etc/profile` 或者 `~/.zshrc`,加入下列内容：

```bash
# kafka
export KAFKA_HOME=/home/sky/work/soft/kafka/kafka
export PATH=$PATH:$KAFKA_HOME/bin
```

通过 `source /etc/profile` 或者 `~/.zshr` 命令载入。


## 最简启动

参考 [Apache Kafka Quick Start](https://kafka.apache.org/quickstart)， 以最简单的方式启动 kafka 并启动 producer 和 consumer：

```bash
cd /home/sky/work/soft/kafka/kafak

# start zookeeper
bin/zookeeper-server-start.sh config/zookeeper.properties

# start kafka server
bin/kafka-server-start.sh config/server.properties

# create topic
bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092

# write events
bin/kafka-console-producer.sh --topic quickstart-events --bootstrap-server localhost:9092

# read events
bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server localhost:9092
```

