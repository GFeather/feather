---
title: MR作业
date: 2018-10-13 10:05:59
tags:
---

## MR作业


###### 配置、提交任务
```
public static void main(String[] args){
    Job job = new Job();
    //Hadoop通过这个类查找包含它的JAR包
    job.setJarByClass(Main.calss);
    //配置作业名称
    job.setJobName("WordCount");
    
    //配置输入、输出文件/目录，输出目录必须不存在以防覆盖其他计算结果
    FileInputFormat.addInputPath(job, new Path("in"));
    FileOutputFormat.setOutputPath(job, new Path("out");
    
    //配置Map、Reduce类
    job.setMapperClass(Mapper.class);
    job.setReducerClass(Reducer.class);
    
    //配置Reduce任务输出类型，Reduce函数的输出要和这里一致
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    
    //waitForCompletion()方法提交作业，参数为true则把进度信息写到控制台，返回值为boolean
    System.exit(job.waitForCompletion(true) ? 0 : 1);
}
```
```
// mr过程
map: (k1, v1) -> list(k2, v2)
combiner: (k2, list(v2)) -> list(k2, v2)
reduce: (k2, list(v2)) -> list(k3, v3)
```
##### MR的横向扩展

###### 1. 数据流
- Hadoop将MR的输入划分为等长的数据块，成为“输入分片”。Hadoop为每个分片分配一个Map任务，实现并行运算。
- Map任务在存储该分片的主机上运行，节省网络带宽，如果存储该分片的所有节点都在运行Map任务，那么需要从其他有空闲Map槽的节点运行。
- Map任务将结果存储在本地而不是HDFS，因为计算任务是动态的，中间结果也是计算的一部分，出错可以很容易再次计算。Reduce的结果存储在HDFS上，提供可靠存储。

###### 2. combiner函数

用来处理Map任务的中间结果，一般为Reduce函数，可以对中间结果进行本地处理，减少网络占用。

###### 输入分片

运行作业的client通过调用`getSplit()`方法计算分片，由InputFormat负责，然后将他们发送给appMaster，appMaster将使用分片信息来调度map任务。

map任务把输入分片传给InputFormat的`createRecordReader()`方法来获得这个分片RecordReader，RecordReader作为记录的迭代器，map任务使用RecordReader生成记录键-值对然后传给map函数。

`getSplit()`通过`maxsize`(默认为Long.max),`minsize`(默认为0)，`blocksize`三个量来计算分片，`max(minsize ,min(blocksize, maxsize))`,所以需要分片大于128M设置`minsize`,小于128M设置`maxsize`,如果当128M*1.1>剩余块大小>128M时，剩余块会作为一个分片。

###### 分区

map之后会进行shuffle，期间进行分区，并按照分区将中间结果分发给reduce任务。分区默认算法按照序列化对象的`compareTo()`方法来分区，可以继承`Patitioner`来自定义分区函数。

##### shuffle和排序

{% asset_img timg.jpg %}

###### Map端

在Map端中间结果并不是直接写道磁盘，出于效率的考虑会现在内存中预排序。当缓冲区内容达到阈值，内容就被后台线程溢出(spill)到磁盘。写磁盘前，线程会根据键来分区，在每个分区中后台线程进行排序，如果由combiner函数，就以排序后的输出作为输入处理。

每次缓冲区到达阈值都会产生一个spill file，在任务完成之前溢出文件会被合并为一个已分区、排序的输出文件，如果至少存在3个spill file conbiner函数就会在写到磁盘前再次运行。

缓冲区数据结构：环形byte[]
1. 原始数据
2. 元数据，描述原始数据在缓冲区的组织形式（因为原始数据存在byte[],每条数据长短不一致，需要记录每一条map输出在缓冲区的位置信息）
    1. 分区编号
    2. key的起始位置
    3. value的起始位置
    4. value的长度

map端shuffle流程：
1. 中间数据在缓冲区中预排序，(根据原始数据对元数据排序)
2. 达到阈值，对数据分区，对每个分区进行排序(*因为分组在排序之后，且按顺序分组，所以自定义的分组键必须在排序键之中*)
2. conbiner函数对输出处理
3. 合并spill file，进行分区、排序，如果至少存在3个spill file再次运行conbiner函数
4. 将合并结果写入磁盘

###### Reduce端

当map任务完成后，reduce的复制线程就会复制map输出文件，可以配置reduce并行复制的线程数。

如果map输出很小，就会直接复制到reduce任务的Jvm中，(配置文件可以制定该缓冲区大小)否则，map输出文件复制到磁盘。一旦达到内存缓冲区阈值或达到map输出阈值，则合并后溢出到磁盘。如果制定combiner，则在合并期间运行降低数据量。

复制完所有map输出后，进入合并阶段，这个阶段将合并map输出，维持其顺序排序。这个阶段是循环进行的，每次合并多个文件，最后阶段即reduce阶段，数据直接输入reduce函数，不用写入磁盘。

map阶段和reduce阶段的shuffle都是按照键类型的`complareTo()`方法来分区、排序，在reduce函数中迭代values时，key也会随着迭代器来迭代。

reduce端shuffle流程：
1. 后台线程复制map输出
2. 复制到Jvm内存或磁盘
3. 达到阈值合并溢出到磁盘，并执行combiner函数
4. 分轮次合并map输出，直接将合并的map输出输入reduce函数

###### 配置调优


##### 运行机制

###### 作业提交

由应用程序调用`Job`的提交方法，首先会检查输入数据是否合法(数据类型、数据是否可以分片)，然后向资源管理器提交作业

###### 作业初始化

资源管理器会将请求传递给调度器，调度器分配一个容器，资源管理器在节点管理器的管理下创建`ApplicationMaster`，为了管理应用进度，`ApplicationMaster`以创建簿记对象来初始化，然后会根据客户端计算的分片来分配map、reduce任务，并在此时分配任务ID。由作业大小不同而决定在appmaster的Jvm中执行或者新创建容器，前者乘坐Uber任务（有名的顺风车）

###### 任务分配

如果不能作为Uber任务则为所有map任务和reduce任务分配容器，首先为map任务分配，当一定数量map任务完成后才为reduce分配容器，在reduce的排序阶段前完成所有map任务。

###### 任务执行

当资源管理器的调度器为任务分配一个容器后，appmaster就会与节点管理器通信启动容器。该任务将由主类为YARNChild的java应用程序执行，因为任务在独立的Jvm中，所以应用代码的bug不会影响到节点管理器

###### 进度和状态的更新

任务在运行时，会对进度保持追踪，并通过umbilical接口与父appmaster通信，每隔3s报告进度和状态（包括计数器的值）

###### 作业的完成

appmaster收到最后一个任务完成的通知后，会将作业状态设置为成功。Job轮询到这个状态后，从`waitForCompletion()`方法返回。appmaster和容器会进行清理，作业信息会被存档。

##### 失败

###### 任务运行失败

一般由于代码抛出异常，退出前Jvm会向父appmaster发送错误报告，错误报告会被记入用户日志，appmaster会将任务标记为失败，释放容器在其他地方重试。其他Jvm、超时处理方式相同

###### application master运行失败

appmaster失败会同样会进行重试，但是在进行两次失败后就会放弃，标记作业失败，可以通过yarn配置增加重试次数。
恢复过程：资源管理器会在新的容器中重新开始一个新的master实例，对于MR任务会通过历史作业来恢复，免于运行已完成任务，也可以关闭。

客户端向appmaster轮询进度，当appmaster运行失败，客户端就需要重新定位新的实例。

###### 节点管理器运行失败

当节点管理器由于崩溃或运行十分缓慢而十分钟内没有心跳，资源管理器就会通知停止心跳的节点管理器，并将它移除节点池以调度启用容器。

失败的节点管理器上运行的任务或appmaster都使用前两节中各自的机制恢复。对于在失败的管理节点上运行的未完成作业的任务，appmaster会安排重新运行，因为他们的中间结果存储在失败的节点上可能无法被访问。

如果应用的运行失败次数过多那么该节点管理器可能会被拉黑

###### 资源管理器运行失败

默认配置中资源管理器存在单点故障，采用双机热备份来获得高可用，所有应用程序的状态信息都存在高可用的状态存储区(ZooKeeper或HDFS备份)。节点管理器信息没有存在状态存储区，因为节点管理器发送第一个心跳时，节点管理器信息很快就会被重构。因为任务由appmaster管理，所以任务信息也不存在状态存储区。

新资源管理器启动后，从状态存储区恢复所有应用程序信息，并为所有应用程序重启appmaster


