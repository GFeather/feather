---
title: Hadoop体系-HDFS
date: 2018-09-20 22:14:27
tags: [Big Data, Hadoop, HDFS]
categories: Hadoop
respository: git@github.com:Gfeather/feather.git
branch: master
---

*Data-driven production value*

## HDFS

*版本：HDFS1.0*

{% asset_img artic.png 简单分布式文件系统架构 %}


#### 需求

- 与单机FS相同或类似的用户(客户端)接口：能够通过shell、程序读写文件
- 多用户(客户端)读写：能够使处理多个读写请求
- 数据可靠性：能够处理数据出错、数据丢失等
- 集群可用性：能够在机器随时损坏的情况下，保证集群、数据可用
- 集群可伸缩性：能够随时增加、减少节点，同时数据相应伸缩

#### 体系结构

{% asset_img hdfs_detail.png %}

从名称节点的命名空间获得位置信息，从数据节点进行读写

> 名称节点：存储元数据信息，核心`FsImage`、`EditLog`，`FsImage`用于维护文件树的所有元数据，`EditLog`记录了所有文件创建、删除、重命名等操作。名称节点会在启动时将`FsImage`载入内存，执行`EditLog`中的操作，期间处于安全模式：只允许读，不允许写。
> 第二名称节点：定时将名称节点的`FsImage`、`EditLog`拉入本地，进行初始化。在这个期间名称节点会将新的操作写入`EditLog.new`中，第二名称节点初始化完成的`FsImage`替代旧的`FsImage`，`EditLog.new`替代`EditLog`。


- 命名空间管理：名称节点管理命名空间，包含目录、文件和块，负责它们的创建、复制和删除
- 通信协议：所有的HDFS通信协议都基于TCP/IP协议
> 客户端——名称节点：通过TCP建立连接，使用客户端协议交互
> 名称节点——数据节点：使用数据节点协议交互
> 数据节点——客户端：通过RPC(Remote Procedure Call)实现交互
	
*1.0局限*：
- 命名空间限制，名称节点能够存储的元数据有限，限制集群存储文件数量。
- 性能瓶颈，系统吞吐量被名称节点吞吐量限制。
- 隔离性，只有一个命名空间，所有应用都可以访问全部文件。
- 可用性，名称节点存在单点故障问题，secondarynode冷备无法热切换。



#### 存储原理

保证数据可靠性，使用多副本冗余、复制机制。副本数量一般为3(或指定更多)副本，系统会维持副本存活数量。

实现：
1，并行访问
2，多副本冗余检错
3，可靠性

- 存放策略：文件副本块分布，本机器一份，本机架其他机器一份，相邻机架一份，更多副本随机分配
- 读取策略：就近读取
- 复制策略：文件会在客户端切分成块，并逐块发送，数据节点根据节点列表进行流水复制，数据写入和复制实现并行。
- 错误与恢复：
	- 名称节点出错：核心数据结构FsImage和EditLog失效则集群失效，由第二名称节点接替，同时从副本恢复
	- 数据节点出错，从副本恢复
	- 数据出错，校验码检错，从副本恢复

#### 数据读写

{% asset_img hdfs_r.png  %}

读：
1. 客户端通过`FileSystem.open()`打开文件，获得输入流`FSDataInputStream`，而对HDFS来说具体的输入流是`DFSInputStream`。
2. `DFSInputStream`的相应的get方法会获得存储某数据块的距离客户端最近的数据节点地址，`FileSystem`会使用它初始化`FSDataInputStream`。
3. 客户端使用`FSDataInputStream.read()`读取文件，当数据块读完之后，`DFSInputStream`会断开与数据节点的连接，获取下一个块，读取结束调用`FSDataInputStream.close()`。

{% asset_img hdfs_w.png  %}

写：
1. 客户端调用`FSDataOutputStream.write()`来向HDFS中的文件写入数据。
2. `FSDataOutputStream`中的数据会被分包存在`DFSOuputStream`的内部队列里，在发送时打成数据包发送，`FSDataOutputStream`向名称节点申请存放数据和副本的数据节点列表，然后这些节点进行流水复制。
3. 为了保证数据准确每个数据节点收到数据包后需要发送'确认包'(ACK),`ACK`会从数据节点管道往前发送直到客户端。
4. 客户端调用`close()`方法后客户端不会写入数据，`DFSOuputStream`会在收到所有分包的`ACK`后通知名称节点关闭文件