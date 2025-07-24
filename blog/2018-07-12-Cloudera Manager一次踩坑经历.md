---
slug: cloudera-manager-pitfall-experience
title: Cloudera Manager一次踩坑经历
authors: [ledai]
tags: [学习历程]
date: 2018-07-12
---
<h1>Cloudera Manager 安装kafka</h1>
   目前大数据生态圈的集群管理平台 目前主流的两个 Cloudera Manager，Ambari 

<!-- truncate -->

Cloudera Manager商业化,平台扩展性强，功能性多。 Ambari 保留原生态，对hadoop 系统入侵性底 ,公司刚搭建的CDH集群，只部署了
一些基本组件 hdfs，yarn，spark，hbase 因为目前项目遇到实时性数据的问题，讨论到kafka消息队列，因为对kafka 不是非常了解，所以想
在集群内加入kafka，之前用的cdh忘记了版本，我记得当时是不支持添加kafka服务的，看到当前系统cm5 cdh5.11 有对kafka的支持，但是有
红字警告,描述如下:`	
Apache Kafka is publish-subscribe messaging rethought as a distributed commit log. Before adding this service, ensure that either the Kafka parcel is activated or the Kafka package is installed.`

<!-- truncate -->
大体的意思是需要确保激活kafka组件 再添加服务，不是很懂，经过一番百度，大多都是使用手动使用Parcel 安装激活的方式，这里提示一定要版本对应，
首先公司的CDH是通过在线安装的，所有服务通过在线rpm安装方式，CDH5.11 对应kafka为kafka2.1.x  下载对应的parcel包后按照流程，分配，激活，没有问题
激活完毕后 奇怪的事情发生了，在服务添加后，初始化服务启动时，报了cdh 以及各种hdfs spark hive服务未激活，直接傻眼了。wtf 在转眼看cdh的时候 之前安装的服务全部挂掉，
再看cdh的版本 什么情况自动从cdh5.11 降到了cdh5.0 这时候又出了一个问题，集群内一台节点无响应，对任何命令全部响应超时，刚开始没有发现...然后这时候我赶紧想通过cdh集群
升级的方式将 环境重新升级到5.11 之后的结果可想而知，一台机器无响应，加上服务的问题， 集群被搞得爆炸...  最终只能通过重装cdh的方式来解决问题(幸好还没有数据介入)
目前的方式只能手动安装kafka 自己管理，这样可以避免影响cdh的工作。

在公司大数据交流会上，提到kafka的数据丢失问题，提到某金融公司一年能丢近20w的订单量，这是非常恐怖的。。还有公司的爬虫组爬取数据也经常丢失，就连ack配置-1 都没有解决问题，
初步问题定位可能在jvm gc  以及集群稳定性问题上，还有之前的一次面试提到的kafka消费完整性，可用性上的问题自己也搞得不是非常透彻，后续会通过学习一一解决这些问题。

这里附上kafka 以及cdh对应安装文档地址

https://www.cloudera.com/documentation/enterprise/release-notes/topics/rn_consolidated_pcm.html#pcm_kafka

kafka中文文档地址

http://kafka.apachecn.org/documentation.html