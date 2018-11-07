---
title: Kafka
date: 2018-11-07 19:54:53
tags: [Kafka]
categories: Hadoop
respository: git@github.com:Gfeather/feather.git
branch: master
---

*Data-driven production value*


## Kafka

kafka是一个分布式流处理平台
1. 具有消息队列功能，可以发布-订阅流式记录
2. 可以实时存储流式记录

##### Kafka的架构

Kafka的架构主要分为三部分
1. Producer //消息的生产，将数据包装为message发送到topic
2. Kafka Cluster    //kafka实例叫做broker，集群由多个broker组成，存储数据，管理多个topic、partition并负载均衡，处理为订阅者发布消息
3. Consumer //消息的订阅，订阅topic消费message。

##### Kafka的存储

Kafka在存储上分为三个层级Topic、Partition、Segment。
Topic: Client角度存储数据的单位，一个Topic存储一类数据
Partition: 一个Topic中数据分为多个分区，一个Partition为一个文件夹。
Segment: 存储message信息，一个Partition中存储多个Segment。

###### kafka数据存储形式

kafka在每个分区内有序，每一条记录都会追加到commit log文件并且分配一个offset来唯一标识。

真实物理存储中每个Partition中的Segment File为一个index file和一个data file组成，他们一一对应，文件名就为第一条记录的offset，index file中存储message在data file中的偏移地址。

###### kafka备份策略

Partition的存储分为leader和follower，leader为主partition，producer写入数据时会先写入leader，再由leader push给follower。leader和follower信息由Zookeeper管理，一旦leader宕机将会选出新leader。

kafka每个partition的多个follower存储在多个broker上，按照轮循算法分配(follower顺序存储在leader之后的broker上，每个partition的leader轮循broker来存储)，这样每个broker上存储的follower、leader数量基本均匀。

##### kafka的消费

Consumer Group包含多个Consumer线程(可能分布在多个机器)，一个Group为逻辑上的订阅者。Topic的每条message会发给所有订阅者一次。

Topic中的partition和Group中的consumer存在消费关系：一个partition只会被一个consumer消费，一个consumer可以消费多个分区。这个消费关系由kafka动态管理，一个consumer消失它的分区就会分给其他consumer，当consumer多于partition就会有consumer线程空闲。


