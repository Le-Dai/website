---
slug: apache-druid-implementation-analysis
title: Apache Druid 实现原理解析
authors: [ledai]
tags: [Druid 源码解析]
date: 2020-03-12
---

#  Apache Druid (Version 0.17)架构 以及 实现原理解析
<!-- truncate -->



- [**Coordinator**](https://druid.apache.org/docs/0.17.0/design/coordinator.html) processes manage data availability on the cluster. 协调器进程管理群集上的数据可用性
- [**Overlord**](https://druid.apache.org/docs/0.17.0/design/overlord.html) processes control the assignment of data ingestion workloads. master进程控制数据提取工作负载的分配。
- [**Broker**](https://druid.apache.org/docs/0.17.0/design/broker.html) processes handle queries from external clients. 代理程序处理来自外部客户端的查询。
- [**Router**](https://druid.apache.org/docs/0.17.0/design/router.html) processes are optional processes that can route requests to Brokers, Coordinators, and Overlords. 路由器进程是可选进程，可以将请求路由到broker，Coordinator和Overlords。
- [**Historical**](https://druid.apache.org/docs/0.17.0/design/historical.html) processes store queryable data. 历史过程存储可查询的数据。
- [**MiddleManager**](https://druid.apache.org/docs/0.17.0/design/middlemanager.html) processes are responsible for ingesting data. MiddleManager进程负责摄取数据。



- **Master**: Runs Coordinator and Overlord processes, manages data availability and ingestion. 运行Coordinator 与 Overlord 进程 协调集群以及数据可用性
- **Query**: Runs Broker and optional Router processes, handles queries from external clients. 运行Broker和Router，处理来自外部客户端的查询。
- **Data**: Runs Historical and MiddleManager processes, executes ingestion workloads and stores all queryable data. 运行Historical和MiddleManager进程，执行提取工作负载并存储所有可查询的数据。
- 

外部依赖

1. 元数据库（Metastore）：存储druid集群的元数据信息，如Segment的相关信息，一般使用MySQL或PostgreSQL
2. 分布式协调服务（Coordination）：为Druid集群提供一致性服务，通常为zookeeper
3. 数据文件存储（DeepStorage）：存储生成的Segment文件，供Historical Node下载，一般为使用HDFS

![druid-architecture](https://druid.apache.org/docs/0.17.0/assets/druid-architecture.png)

