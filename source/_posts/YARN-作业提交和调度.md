---
title: YARN-作业提交和调度
date: 2018-10-17 17:22:25
tags: [YARN]
categories: Hadoop
respository: git@github.com:Gfeather/feather.git
branch: master
---

*Data-driven production value*

## YARN-作业提交和调度

####  Yarn作业提交流程

1. 客户端提交作业到resourcemanager，resourcemanager检查作业分片、数据。
2. resourcemanager返回共享资源路径和JobId
3. 客户端将job.jar、split、job.xml放在共享资源路径下
4. 告诉resourcemanager资源设置完毕
5. resourcemanager分配容器，并在该容器启动appmaster
6. resourcemanager到对应节点上先启动container，然后在container中启动appmaster
7. appmaster到共享资源路径下载资源，split、job.xml
8. appmaster创建簿记对象来管理任务进度，初始化job
9. appmaster向rm申请执行map、reduce任务的资源
10. resourcemanager返回资源节点
11. appmaster到对应节点启动container，在该container中启动任务
12. maptask到共享资源路径下载jar包
13. maptask执行过程中会汇报执行进度和状态
14. 当有一个maptask执行完后，就会启动container启动reducetask任务
15. reducetask到共享路径下载资源，当所有maptask任务执行完毕，开始执行reducetask
16. 当所有任务完成，appmaster就会向resourcemanager申请销毁资源并注销自己

##### 运行机制

client连接ResourceManager，提交运行一个Application master，ResourceManager在一台主机上运行它，application master做什么取决于应用。

###### 应用生命期

YARN的资源请求非常灵活，可以申请各种资源(CPU,内存等)，同时可以在任何时候申请资源。

1. 作业-应用，如MR，分配一个容器执行MR作业，结束之后释放
2. 工作流-应用，如Spark，在分配的容器上按照工作流执行作业，复用容器
3. 守护进程-应用，多个用户共享的长期运行的应用

##### YARN中的调度

1. FIFO

最简单的调度模型，无法用于大型集群
2. 容量调度，定义多个资源量不同的队列来供给相应的任务

配置：可以按照层次树状划分，作业在给定容器中执行。在弹性容器中，当其他容器空闲，当前作业所需资源大于当前容器，可能会将其他容器空闲资源分配给他。

在capacity-scheduler.xml文件中配置，格式为yarn.scheduler.capacity.<queue-path>.<sub-property>,<queue-path>表示队列的层次路径，如root.src

```
<configuration>
    <property>
        <name>yarn.scheduler.capacity.root.queues</name>
        <value>src,dev</value>
    </property>
    <property>
        <name>yarn.scheduler.capacity.root.src.capacity</name>
        <value>40</value>
    </property>
    <property>
        <name>yarn.scheduler.capacity.root.src.maximum-capacity</name>
        <value>50</value>
    </property>
    <property>
        <name>yarn.scheduler.capacity.root.dev.capacity</name>
        <value>60</value>
    </property>
</configuration>
```

队列放置在应用中决定，比如MapReduce中可以用`mapreduce.job.queuename`来指定使用的队列

3. 公平调度，当只有一个任务时，会给它分配所有资源，知道有新任务提交，将平分资源

使用公平调度器需要将yarn-site.xml文件中的`yarn.resourcemanager.scheduler.class`设置为公平调度器的全限定名:`org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler`

配置：通过位于类路径下的fair-scheduler.xml来配置。

```
<allocations>
    <defaultQueueSchedulingPolicy>fair</defaultQueueSchedulingPolicy>
    <queue name="src">
        <wright>40</weight>
        <schedulingPolicy>fifo</schedulingPolicy>
    </queue>
    <queue name="dev">
        <wright>60</weight>
        <queue name="eng" />
        <queue name="sci" />
    </queue>
    <queuePlacementPolicy>
        <rule name="specified" create="false" />
        <rule name="primaryGroup" create="false" />
        <rule name="default" queue"dev.eng" />
    </queuePlacementPolicy>
</allocations>
```
可以使用潜逃来定义层次，所有队列都是root的子队列无论有没有显示嵌套。可以分配资源比例、各队列调度策略。队列默认调度可以通过顶层元素`defaultQueueSchedulingPolicy`进行设置，如果省略，默认为公平调度。策略可以配该队列的`schedulingPolicy`所覆盖

队列放置，公平调度器使用基于规则的系统给来确定应用放置，`queuePlacementPolicy`中指定了一系列规则，将会被依次匹配，`specified`，表示把应用放进指明的队列中。`primaryGroup`，表示把应用放在以用户的主Unix组名命的队列中。`defaul`，表示默认规则。

通过将`yarn.scheduler.fair.preemption`设为true，启用抢占，且要配置相关超时参数

##### 延迟调度

YARN会以本地请求为主(即数据在本地节点的应用)，本地无空闲节点时为应用安排其他容器，

此时如果等待一段时间会增加分配到节点的机会，YARN将心跳信息作为潜在调度机会，当调度机会到达设定的最大调度机会时，才放松本地限制接受下一次调度机会。


容量调度通过`yarn.scheduler.capacity.node-locality-delay`来设置延迟调度错过的调度机会数量。

公平调度器可以通过`yarn.scheduler.fair.locality.threshold.node`来设置调度机会比例，通过`yarn.scheduler.fair.locality..threshold.rack`来设置需要等待的时长。

##### 主导资源公平性

YARN会以应用的主导资源作为调度的度量，即应用需求多的资源类型。

*参考《HADOOP权威指南》*
