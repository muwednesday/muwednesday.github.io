---
title: Kafka-basic
date: 2019-09-30 15:29:54
tags: Kafka
categories: Middleware
---

kafka basic

<!-- more -->

## kafka 安装

```bash
curl -O https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.5.5/apache-zookeeper-3.5.5-bin.tar.gz
tar xvzf apache-zookeeper-3.5.5-bin.tar.gz -C zookeeper --strip-components=1

bin/zkServer.sh start
ps aux | grep zoo.cfg
```

https://www.explainshell.com/
man
tldr
tar -xzf FILENAME -C FOLDER --strip-components=1


```bash
curl -O http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.3.0/kafka_2.12-2.3.0.tgz
tar xvzf kafka_2.12-2.3.0.tgz -C kafka --strip-components=1

bin/kafka-server-start.sh -daemon config/server.properties
```


```
[repositories]
  local
  my-ivy-proxy-releases: http://repo.typesafe.com/typesafe/ivy-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
  my-maven-proxy-releases: http://maven.aliyun.com/nexus/content/groups/public
```

```
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
bin/kafka-topics.sh --list --zookeeper localhost:2181
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning

bin/kafka-topics.sh --create --zookeeper localhost:2181/test --replication-factor 3 --partitions 1 --topic my-replicated-topic
bin/kafka-topics.sh --describe --zookeeper localhost:2181/test --topic my-replicated-topic

bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic

ps aux | grep -E /server.*\.properties