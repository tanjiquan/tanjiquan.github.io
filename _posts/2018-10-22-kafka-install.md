---
layout:     post
title:      Kafka 安装
subtitle:   Kafka
date:       2018-10-22
author:     tanjiquan
header-img: img/in-post/kafka-bg.png    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 消息队列
---

## kafka单机版安装

以下安装过程比较简单，只是一个单机版的操作，集群安装请自行参照官网。
仅用于学习研究。

### （一）zookeeper安装
`1.wget http://apache.fayea.com/zookeeper/zookeeper-3.4.10/zookeeper-3.4.10.tar.gz`

`2.mkdir /Users/tanjiquan/software/hadoop/zookeeper`

`3.tar -zxvf zookeeper-3.4.10.tar.gz -C /Users/tanjiquan/software/hadoop/zookeeper`

`4.cd zookeeper-3.4.10/`

`5.mkdir data`

`6.mkdir logs`

`7.cd conf`

`8.cp zoo_sample.cfg zoo.cfg` 

`9.vi zoo.cfg 配置`
> dataDir=/Users/tanjiquan/software/hadoop/zookeeper/zookeeper-3.4.10/data
dataLogDir=/home/tanjiquan/zookeeper/zookeeper-3.4.10/logs

`10.vi ~/.bash_profile 或者 vi /etc/profile`
> ZOOKEEPER\_HOME=/Users/tanjiquan/software/hadoop/zookeeper/zookeeper-3.4.10
PATH=$ZOOKEEPER\_HOME/bin:$PATH
CLASSPATH=$ZOOKEEPER\_HOME/lib:

`11.source ~/.bash_profile (/etc/profile)`

`12.sh zkServer.sh start`

### （二）kafka安装
`1.wget http://apache.fayea.com/kafka/0.11.0.3/kafka_2.12-0.11.0.3.tgz`

`2.mkdir kafka`

`3.tar -zxvf kafka_2.12-0.11.0.3.tgz -C /Users/tanjiquan/software/hadoop/kafka`

`4.cd kafka/kafka_2.12-0.11.0.3/`

`5.mkdir logs`

`6.cd config`
 
`7.vi server.properties`
> listeners=PLAINTEXT://127.0.0.1:9092
log.dirs=/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/kafka-logs

`13.vi ~/.bash_profile 或者 vi /etc/profile`
> KAFKA_HOME=/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3
PATH=$KAFKA_HOME/bin:$PATH

`14.source ~/.bash_profile (/etc/profile)`

`15.nohup /Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/bin/kafka-server-start.sh //Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/config/server.properties >>/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/kafka.log &`

### （三）kafka常用命令
*  1.启动kafka   
   `/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/bin/kafka-server-start.sh //Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/config/server.properties >>/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/kafka.log &`
*  2.创建topic   
`/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/bin/kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --replication-factor 1 --partitions 1 --topic test_topic`

*  3.删除topic
`/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/bin/kafka-topics.sh --delete --zookeeper 127.0.0.1:2181 --topic test_topic` 

*  4.查看topic列表   
`/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/bin/kafka-topics.sh --list --zookeeper 127.0.0.1:2181`

*  5.查看某个topic信息   
`/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/bin/kafka-topics.sh --describe --zookeeper 127.0.0.1:2181  --topic test_topic`

*  6.生产端cli   
`/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/bin/kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic test_topic` 

*  7.消费端cli   
`/Users/tanjiquan/software/hadoop/kafka/kafka_2.12-0.11.0.3/bin/kafka-console-consumer.sh --topic test_topic --bootstrap-server 127.0.0.1:9092 --from-beginning `
