---
layout:     post
title:      "Hadoop 学习（1）Hadoop 各服务的作用简述"
date:       2018-02-04 20:09:00
author:     "Kent"
header-img: "img/post-bg-2016.jpg"
catalog: true
tags:
    - hadoop
---

[TOC]

注：以下服务为基于安装`Hadoop 2.6.0-cdh5.8.3`后的所得服务，但并不是全部。

## 1. HDFS

### NameNode 

`NameNode`是一个中心服务器，负责管理文件系统的`namespace`以及客户端对文件的访问。

`NameNode`执行文件系统的`namespace`操作，比如打开、关闭、重命名文件或目录。同时它还确定`block`到`DataNode`节点的映射。


**Namenode 详细功能如下：**
- 它是维护和管理 Datanode 的主守护进程；
- 它记录存储在集群中的所有文件的元数据，例如 block 的位置、文件大小、权限、层次结构等。有两个文件与元数据关联：
    - FsImage：它包含自 Namenode 开始以来文件的 namespace 的完整状态；
    - EditLogs：它包含最近对文件系统进行的与最新 FsImage 相关的所有修改。
- 它记录了发生在文件系统元数据上的每个更改。例如，如果一个文件在 HDFS 中被删除，Namenode 会立即在 EditLog 中记录这个操作。
- 它定期从集群中的所有 Datanode 接收心跳信息和 block 报告，以确保 Datanode - 处于活动状态；
- 它保留了 HDFS 中所有 block 的记录以及这些 block 所在的节点；
- 它负责管理所有 block 的复制；
- 在 Datanode 失败的情况下，Namenode 会为副本选择新的 Datanode，平衡磁盘使用并管理到 Datanode 的通信流量。

### DataNode

在集群内部，文件被分成一个或多个`block`，这些`block`被存储在一组`DataNode`中，而`DataNode`则用于管理它所在节点上的存储。

Datanode负责处理文件系统客户端的读写请求。在Namenode的统一调度下进行数据块的创建、删除和复制。

**Datanode 详细功能如下：**
- 这些是丛属守护进行或在每台从属机器上运行的进程；
- 实际的数据存储在 Datanode 上；
- 执行文件系统客户端底层的读写请求；
- 定期向 Namenode 发送心跳报告 HDFS 的整体健康状况，默认频率为 3 秒

NameNode and DataNode 更多请参考：[HDFS Architecture Guide](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html#NameNode+and+DataNodes)
 
### FailoverController

故障切换控制器，运行在`NameNode`旁边，主要包含以下3个功能：

1\) 监控`NameNode`的健康状态（心跳检测）；  
2\) 向`ZooKeeper`定期发送心跳，使自己可以被选举；  
3\) 当自己被`ZooKeeper`选择为主时，通过 RPC 调用使对应的`NameNode`转换为`active`状态。

### JournalNode

用于存储`EditLog`，相关资料如下：

`NameNode`默认采用 Clouderea 公司实现的基于`QJM(Quorum Journal Manager)`的方案来实现其共享存储方案，在`QJM`方案中，`NameNode` 在执行 HDFS 客户端提交的创建文件或者移动文件这样的写操作的时候，会首先把这些操作记录在`EditLog`文件之中，然后再更新内存中的文件系统镜像（内存中的文件系统镜像用于`NameNode`向客户端提供读服务，而`EditLog`则只是在数据恢复的时候起作用）。

QJM 共享存储的基本思想来自于 Paxos 算法 (参见参考文献 [3])，采用多个称为 `JournalNode`的节点组成的`JournalNode`集群来存储`EditLog`，每个`JournalNode`保存同样的`EditLog`副本。每次`NameNode`写`EditLog`的时候，除了向本地磁盘写入 `EditLog`之外，也会并行地向`JournalNode`集群之中的每一个`JournalNode` 发送写请求，只要大多数 (majority) 的`JournalNode`节点返回成功就认为向 `JournalNode`集群写入` EditLog`成功。如果有 2N+1 台 `JournalNode`，那么根据大多数的原则，最多可以容忍有 N 台`JournalNode`节点挂掉。

更多请参见：[Hadoop NameNode 高可用 (High Availability) 实现解析](https://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-name-node/)

### Balancer

用于平衡集群之间个节点的磁盘利用率。

因为 HDFS 的数据可能并不总是被均匀地分布的在`DataNodes`中，比如添加新的`DataNode`到现有的集群，所以 HDFS 提供了`Balancer`服务用于分析`block`的放置并平衡`DataNode`中的数据。`Banlancer`会移动`block`直到集群被认为是平衡的，这也意味着每个`DataNode`的利用率（节点上已用空间与节点总容量的比率）与集群的利用率（集群的已用空间与集群的总容量的比率）相差不会超过给定的阈值百分比。

更多请参见：[HDFS Balancers](https://www.cloudera.com/documentation/enterprise/5-7-x/topics/admin_hdfs_balancer.html)

### HttpFS

提供 HTTP 方式访问 HDFS 服务的功能。它支持所有 HDFS filesystem 操作（包括读取与写入）的 REST Api。 

更多请参见：[Adding HttpFS
](https://www.cloudera.com/documentation/enterprise/5-7-x/topics/admin_httpfs.html)

## 2. HBase

### Master

负责监视集群中的所有 RegionServer 实例，并且是所有元数据更改的接口。在分布式集群中，Master 通常运行在 NameNode 之上。其具体提供的功能如下：

**1) 提供的接口**

HMasterInterface 主要提供面向元数据的方法：
- Table (createTable, modifyTable, removeTable, enable, disable)
- ColumnFamily (addColumn, modifyColumn, removeColumn)
- Region (move, assign, unassign)

例如，当 HBaseAdmin 方法 disableTable 被调用时，它由 Master Server 提供服务。

**2) 运行的线程**

Master 运行了几个后台线程，其中包括:
- LoadBalancer: 定期的，当没有 region 被 transition 时，LoadBalancer 将运行并移动 region 以平衡集群的负载；
- CatalogJanitor：定期检查并清理 .META. 表；

详情请参见：[9.5 HBase Master](http://hbase.apache.org/0.94/book/master.html)

### RegionServer

负责服务和管理 regions。在分布式集群中，RegionServer 运行在 DataNode 之上。其具体的功能如下：

**1) 提供的接口**

HRegionRegionInterface 公开的方法包含面向数据和 Region 维护的方法：
- Data (get, put, delete, next, etc.)
- Region (splitRegion, compactRegion, etc.)

例如，当调用 HBaseAdmin 的方法 majorCompact 操作表时，客户端实际上会遍历指定表的所有区域，并直接向每个 region 请求 major compaction 操作。

更多请参见：[9.6 HBase RegionServer](http://hbase.apache.org/0.94/book/regionserver.arch.html)

## 3. Yarn

### NodeManager
 
负责启动和管理节点上的 containers。Containers 则用于执行 AppMaster 指定的任务。

### ResourceManager (RM)

负责跟踪与管理集群中的资源，并调度运行在 Yarn 上的 applications（例如 MapReduce 作业）。

在 Hadoop 2.4 之前，RM 具有单点故障，实现高可用请参考：[Yarn ResourceManager High Availability](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/ResourceManagerHA.html)

### JobHistory

以 REST API 的方式向用户提供完成的 applications 的状态查询。目前它只支持 MapReduce 并提供完成的 job 信息。

## 4. Spark_on_yarn

### HistoryServer

显示有关完成的 Spark aplications 的历史记录信息。

## 5. Hive

### HiveServer2

使客户端能够对 Hive 执行查询功能。HiveServer2 是已弃用的 HiveServer1 的继承者。它支持多客户端并发和身份验证，旨在为 JDBC 和 ODBC 等开放 API 客户端提供更好的支持。

HiveServer2 是作为组合服务运行的单个进程，其中包括基于 Thrift 的 Hive 服务（TCP或HTTP）和用于 Web UI 的 Jetty Web 服务器。

更多请参见：[Hive HiveServer2 Overview](https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Overview)

### HiveMetaStore

将 Hive 表和分区的元数据存储在关系型数据库中，并提供 API 访问表和分区的所有元数据。


更多请参见：
- [Hive AdminManual MetastoreAdmin](https://cwiki.apache.org/confluence/display/Hive/AdminManual+MetastoreAdmin#AdminManualMetastoreAdmin-Introduction)
- [Hive Configuring the Hive Metastore](https://www.cloudera.com/documentation/enterprise/5-8-x/topics/cdh_ig_hive_metastore_configure.html)

### GateWay

## 6. Zookeeper

### Server

其主要功能如下：

- 基于 Paxos 算法提供 **Master 选举**与**主备切换**功能； 
- HBase 通过创建并监听 ZK 节点，实现**系统容错**功能；
- HBase 通过在 ZK 上记录与感知 RootRegion(region 元数据)的变化，来实现**管理 RootRegion**；
- HBase 将 Region 的上下线和重新上下线的状态管理/通知交由 ZK； 管理，来实现**Region 管理**；
- HBase 通过在 ZK 上创建 SplitWAL 节点，并由 ZK 担负相互通知和信息持久化的角色，从而实现**分布式的 SplitWAL 任务管理**；
- 在 Yarn 中，ZK 用来**存储应用的运行状态**；
- ...

## 7. mgmt

- ActivityMonitor
- AlertPublisher
- ClockOffset
- EventServer
- HostMonitor
- ServiceMonitor


## 参考链接

- [HDFS Architecture Guide](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html#NameNode+and+DataNodes)
- [HDFS Issue: HA: HDFS portion of ZK-based FailoverController](https://issues.apache.org/jira/browse/HDFS-2185?attachmentSortBy=dateTime)
- [HDFS Namenode HA原理详解](https://www.cnblogs.com/shenh062326/p/3870219.html)
- [HDFS Hadoop NameNode 高可用 (High Availability) 实现解析](https://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-name-node/)
- [HDFS Balancers](https://www.cloudera.com/documentation/enterprise/5-7-x/topics/admin_hdfs_balancer.html)
- [HDFS Adding HttpFS
](https://www.cloudera.com/documentation/enterprise/5-7-x/topics/admin_httpfs.html)
- [9.5 HBase Master](http://hbase.apache.org/0.94/book/master.html)
- [9.6 HBase RegionServer](http://hbase.apache.org/0.94/book/regionserver.arch.html)
- [Yarn NodeManager](https://hadoop.apache.org/docs/r2.8.0/hadoop-yarn/hadoop-yarn-site/NodeManager.html)
- [Yarn ResourceManager Restart](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/ResourceManagerRestart.html)
- [Yarn ResourceManager High Availability](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/ResourceManagerHA.html)
- [Yarn History Server REST API's](https://hadoop.apache.org/docs/r2.4.1/hadoop-yarn/hadoop-yarn-site/HistoryServerRest.html)
- [Spark_on_yarn Managing the Spark History Server](https://www.cloudera.com/documentation/enterprise/5-6-x/topics/admin_spark_history_server.html)
- [Hive HiveServer2 Overview](https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Overview)
- [Hive AdminManual MetastoreAdmin](https://cwiki.apache.org/confluence/display/Hive/AdminManual+MetastoreAdmin#AdminManualMetastoreAdmin-Introduction)
- [Hive Configuring the Hive Metastore](https://www.cloudera.com/documentation/enterprise/5-8-x/topics/cdh_ig_hive_metastore_configure.html)
- [ZooKeeper 原理及其在 Hadoop 和 HBase 中的应用](http://blog.jobbole.com/110388/)