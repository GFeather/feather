---
title: Zookeeper
date: 2018-10-26 22:32:51
tags: [Zookeeper]
categories: Hadoop
respository: git@github.com:Gfeather/feather.git
branch: master
---

*Data-driven production value*

## Zookeeper
Zookeeper是一个分布式一致性协调系统，提供分布式一致性能力，可以维护多个分布式一致的分布式状态。基于这个能力来解决分布式系统中遇到的大量分布式一致性问题。核心在于利用zookeeper维护的全局一致的状态来进行多种表述。

##### zookeeper的安装：
1. 解压
2. 配置环境变量
3. 创建配置文件zoo.cfg
```
//在末尾添加zookeeper集群主机
server.id=主机名：2888:3888
```
4. 在zoo.cfg的datadir目录下创建myid文件，编辑为主机的id

##### 文件系统
zookeeper内的数据以节点存储，结构为目录树，只有绝对路径，没有目录概念
节点分类：
1. 生命周期：
    1. 临时节点    EPHEMERAL    `create    -e    node_name    'value'`
        临时节点不能有子节点，且客户端关闭就会删除
    2. 永久节点    `create    node_name    'value'`
2. 有无编号：
    1. 有编号    SEQUENTIAL    `create    -s    [-e]    node_name    'value'`
    2. 无编号
tips：zk的Znode存储大小不要超过1M，最好不超过1kb。对于zk，有几个节点数据就会存储几份

##### 监听机制：
监听事件的类型：
    Znode创建 nodeCreated
    Znode删除	nodeDeleted
    Znode的数据变化 nodeDataChanged
    Znode的子节点的变化 nodeChildrenChanged
{% asset_img watch.png %}
监听的添加操作：
    shell 命令最后的watch 代表添加了监听
    ls        ---nodeChildrenChanged
    stat path [watch]    所有信息
    get        ---nodeDataChanged


#####  节点状态信息
{% asset_img stat.png %}
其中的全局id标识：
id标识全局的事件提交顺序
    cZxid：创建节点事件的id标识
    mZxid：修改节点事件的id标识     
    pZxid：子节点变化的事件的id

##### API
```
//获取zookeeper连接
new ZooKeeper("hadoop01:2181", 5000, null);
//创建节点
/**
   * 参数1：节点路径 /开始 string
   * 参数2：节点内容 byte[]
   * 参数3：权限 默认  
   * 参数4：节点的类型
   */
zk.create("path",  "value".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
//修改节点内容
  /**
   * 参数1 节点路径
   * 参数2 修改内容
   * 参数3 数据版本 如果不知道 -1 代表获取最新版本
   */
zk.setData("path", "value".getBytes(), -1);
//查看节点内容
  /**
   * 参数1：节点路径
   * 参数2：监听 不要监听 null
   * 参数3：节点状态对象 不知道 null
   */
byte[] content=zk.getData("path", null, null);
//查看子节点
  /**
   * 参数1：节点路径
   * 参数2：监听
   */
List<String> children = zk.getChildren("/", null);
//返回值 是节点状态对象 封装节点的状态信息的 当前节点不存在 返回null
Stat exists = zk.exists("/bd1807", null);
```

####    Zookeeper的应用

##### 数据发布/订阅
zookeeper的发布/订阅系统采用Push/Pull结合的模式，即Zookeeper通知客户端事件发生，客户端去拉取新数据
通过在Zookeeper上存储需要大量共享的配置信息，通过Zookeeper来进行动态的配置管理

##### 负载均衡
将服务器信息存储在zookeeper上，当负载变化时改变服务对应的服务器节点数

##### 命名服务
使用zookeeper的顺序节点来生成分布式环境下唯一的名称

##### 分布式协调/通知
通过使用zookeeper的Watch机制来实现，分布式、不同系统间任务流程、事件的协调
###### 任务注册
对应任务进程启动时会创建一个相关任务节点，如果以存在则代表该任务已经有节点执行
###### 任务热备份
实现热备份容灾，将一个任务部署到多个机器上，都在zookeeper上注册临时有序节点，约定序号最小节点为RUNNING主机，当主机宕机临时节点消失，zookeeper通知其他主机，当其他主机发现自己序号最小时将运行状态变为RUNNING。
###### 冷备切换
与热备不同的是首先将冷备任务分组，冷备主机会遍历当前组下所有Task，当发现没有子节点任务时，就在这个任务下添加子节点，如果有多个主机创建则依然“最小序号优先”，而其他节点将删除创建的节点继续遍历冷备列表。
冷热对比：热备提供了多个节点实时接替继续提供服务，冷备则是由空闲主机来执行相关任务
###### 心跳检测
创建临时节点，通过判断节点是否存在来判断是否存活
###### 工作任务汇报
各个任务会实时将任务进度写到节点
###### 系统调度
通过改变zookeeper节点的值来控制客户端执行相应的业务逻辑

##### Master选举
可以让客户端集群在zookeeper上创建相同节点，创建成功的就是Master，同时zookeeper还可以实现实时监听Master存活

##### 分布式锁

排它锁：写锁，保证当前只能有一个事务获得锁，可以通过创建固定节点来实现
共享锁：读锁，可以有多个客户端同时读取，保证只有在没有共享锁的时候才进行写操作

##### 分布式队列
###### FIFO
1. 通过调用`getChildren()`接口来获得节点下所有子节点，即队列中所有元素
2. 确定自己的节点在所有子节点的顺序
3. 如果自己不是序号最小的节点就进入等待，同时向比自己小的最后一个节点注册Watch
4. 收到通知后重复步骤1
