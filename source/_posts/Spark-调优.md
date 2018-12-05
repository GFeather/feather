---
title: Spark 调优
date: 2018-12-05 17:33:16
tags: [spark]
categories: Spark
respository: git@github.com:Gfeather/feather.git
branch: master
---

*Data-driven production value*


## Spark-调优

##### 开发调优

- 避免创建重复RDD
- 尽可能复用同一个RDD
- 对多次使用的RDD进行持久化
- 尽量避免使用shuffle类算子
- 使用map-side预聚合的shuffle操作
- 使用高性能的算子
    -  使用reduceByKey/aggregateByKey替代groupByKey
    -  使用mapPartitions/foreachPartitions替代普通map/foreach  ，   能够提升效率，但是一个task处理整个分区可能会OOM
    -  使用filter之后进行coalesce操作
    -  使用repartitionAndSortWithinPartitions替代repartition与sort类操作 ，    如果重新分区后需要排序，可以在分区过程排序
- 广播大变量
- 使用Kryo优化序列化性能
- 优化数据结构
- 提高并行度
    - 使每个core处理2~3个任务，即task数为application申请core数量的2~3倍
    - 创建RDD时手动在参数中设置
    - sparkConf或者Spark-submit中指定`spark.default.parallelism`
- 数据本地化
    - 对spark性能有巨大影响，尽量在数据所在节点的executor上执行task
    - 本地化级别
        - PROCESS_LOCAL：数据和计算它的代码在同一个JVM进程中。
        - NODE_LOCAL：数据和计算它的代码在一个节点上，但是不在一个进程中，比如在不同的executor进程中，或者是数据在HDFS文件的block中。
        - NO_PREF：数据从哪里过来，性能都是一样的。
        - RACK_LOCAL：数据和计算它的代码在一个机架上。
        - ANY：数据可能在任意地方，比如其他网络环境内，或者其他机架上。
    - spark优先在数据本地执行task，没有空闲executor的话spark会等待一段时间。可以通过`spark.locality.wait`等配置

##### 资源参数调优

###### 目的

- 设置合适的executor数量
- 给executor分配合适的task数量、core数量、内存数量
- 配置合适的spark存内存策略

###### 参数

- num-executors
    - 设置作业的executor数量，影响task执行并行度
    - 根据资源队列来确定，确定作业的并行度下限
- executor-memory
    - 每个executor的内存，决定task运行内存
    - 根据资源队列来确定，单个executor可以分配多少，以及考虑一个executor上同时执行的task需要多少
- executor-cores
    - 每个executor的core数，决定task执行并行度
    - 根据资源队列来确定，决定同时执行的task数量，考虑单个executor的内存和task需要内存的关系
- driver-memory
    - 决定driver进程内存
- spark.default.parallelism
    - 每个stage的task数量，决定并行度
    - 保证资源利用率，大概为cores的2~3倍
- spark.storage.memoryFraction
    - RDD持久化数据在Executor内存中的占比，默认0.6
    - 作业中持久化使用较多则设置大一些，否则数据会写入磁盘
- spark.shuffle.memoryFraction
    - shuffle过程中一个task拉取到上一个stage的task的输出后进行聚合时能够使用的Executor内存占比，默认0.2
    - 如果RDD持久化操作较少，shuffle操作较多，可以提高shuffle内存降低RDD持久化内存

##### Shuffle调优

- spark.shuffle.file.buffer
    - 默认值 32k
    - shuffle write task写入磁盘前的缓冲区大小，内存充足的条件下可以适当增加用以减少写入磁盘次数。合理调节可以获得1%~5%增益
- spark.reducer.maxSizeInFlight
    - 默认值 48M
    - shuffle read task的缓冲区大小，决定每次能拉取多少数据，内存充足的条件下可以适当增加用于减少网络传输次数。合理调节可以获得1%~5%增益
- spark.shuffle.io.maxRetries
    - 默认值 3
    - shuffle read task拉取失败重试次数，可以提高稳定性
- spark.shuffle.io.retryWait
    - 默认值 5s
    - 每次重试拉取的等待间隔，增大间隔可以提高稳定性
- spark.shuffle.memoryFraction
    - 默认值 0.2
    - Executor中分配给shuffle read task进行聚合操作的内存，调高以避免shuffle read操作频繁读写磁盘。合理调节可以提高10%增益
- spark.shuffle.manager
    - 默认值 sort
    - ShuffleManager的类型，三个可选项:hash、sort和tungsten-sort。SortShuffleManager会对结果排序，如果不需要则通过bypass机制或优化的HashShuffleManager
-  spark.shuffle.sort.bypassMergeThreshold
    - 默认值 200 
    - 当ShuffleManager为SortShuffleManager时，如果shuffle read task的数量小于这个阈值（默认是200），则shuffle write过程中不会进行排序操作，而是直接按照未经优化的HashShuffleManager的方式去写数据，但是最后会将每个task产生的所有临时磁盘文件都合并成一个文件，并会创建单独的索引文件。
- spark.shuffle.consolidateFiles
    - 默认值 false
    - 设置为true会合并shuffle write的文件，如果不需要排序，可以使用HashShuffleManager开启合并。

##### 数据倾斜

- 使用Hive ETL预处理数据
	- 场景: 数据在Hive表里，经常使用spark进行计算
	- 方案: 使用Hive预处理，使其变成不倾斜
	
- 过滤少数导致倾斜的key
	- 场景: 导致倾斜的Key只有少数几个
	- 方案: 将倾斜Key和正常数据分开处理，处理倾斜Key时增加并行度，最后将结果合并

- 提高shuffle操作的并行度
	- 场景: 面对必须处理的倾斜数据，最简单的方案
	- 方案: 设置spark.sql.shuffle.patitions提高map task并行度，增加分区数，来处理倾斜数据

- 两阶段聚合（局部聚合+全局聚合）
	- 场景: 使用聚合类算子时遇到数据倾斜
	- 方案: hash生成n种(想要分区数)字符串随机添加在可能数据倾斜的字段前，将数据分为均匀的几块进行聚合，再去掉前缀聚合。分两阶段聚合

---
*Join*

- 将reduce join转为map join
	- 场景: 大表 * 小表
	- 方案: 将小表设置为共享变量
	- 如果: 大表 *  大表 
		有两种方式：
		1、把其中一个较小的大表拆分成很多的小表，然后执行很多mapjoin
		2、先过滤，然后再做操作。一般来说， 原来不能使用mapjoin的操作， 很有可能在
		进行了过滤之后， 就能把不用参与join的数据给过滤掉
		
		使用mapjoin解决数据倾斜的原理，就是直接把shuffle直接舍弃


- 采样倾斜key并分拆join操作
    - 场景: Join时有少数几个Key倾斜
    - 方案: 通过采样来确定倾斜key，然后将倾斜key提取出来，添加随机n种前缀，对要Join的RDD中相同的Key添加n种前缀扩容n倍，然后倾斜Key连接、正常Key连接, 最后将结果union

- 使用随机前缀和扩容RDD进行join
	- 场景: 大多数Key数据量都很大
	- 方案: 类似上一方案，将左右key都添加随机前缀，将要Join的RDD进行全量扩容
			
- 多种方案组合使用


