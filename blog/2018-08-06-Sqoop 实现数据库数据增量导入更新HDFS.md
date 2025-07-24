---
slug: sqoop-incremental-data-import-hdfs
title: Sqoop 实现数据库数据增量导入更新HDFS
authors: [ledai]
tags: [学习历程, sqoop, 数据增量同步]
date: 2018-08-06
---

<h1>Sqoop 实现数据库数据增量导入更新HDFS</h1>

   首先我们来讨论一下增量同步需要注意的那些问题:
<!-- truncate -->

1.数据既然是增量同步当然是只同步变化的数据 不是全量拉取覆盖操作 时间以及资源浪费问题

2.有新数据insert 需要同步 当然update 更新的数据也需要同步 需要注意的是更新的数据同步问题

3.并发控制问题

<h2>目前大部分的mysql数据同步都是通过监听bin/log 实现 现在我们使用sqoop 来实现mysql增量同步hdfs</h2>


<h3>1.安装sqoop</h3>
 这里比较简单 就不多说了  我使用的是CDH 直接添加服务即可 下一步下一步就完成，如果是手动安装需要到sqoop官网下载对应tar包解压配置这里就不
 
 多说了  自行百度
 
<h3>2.sqoop的配置解释</h3>

 先直接贴出命令example 
 
```
sqoop import \

--connect jdbc:mysql://***:3306/disaster \

--username ** \

--password ** \

--table sqoop_test \

--check-column add_time \

--incremental lastmodified \

--last-value "2018-08-06 12:07:15" \

--merge-key id \

-m 1 \

--target-dir /user/sqoop-test \

 ```

1、Append方式

参数	说明

–incremental append	基于递增列的增量导入（将递增列值大于阈值的所有数据增量导入Hadoop）

–check-column	递增列（int）

–last-value	阈值（int）

append模式增量导入  大概使用方法 check-column 对指定的列进行判断  大于则增量导入 例如 -check-column ID  –last-value 5
则ID大于5的ID会被全部导入  但是有一个问题  ID 为4的数据被更新的怎么办? 无法实现更新数据的同步 

2、lastModify方式 
此方式要求原有表中有time字段，它能指定一个时间戳，让Sqoop把该时间戳之后的数据导入至 hdfs(这里为什么没有写hive  后续就知道了)
正因为之前数据会出现更新的问题所有有了这个时间戳time 字段也就是我命令里面的add_time 正规一点将应该讲update_time 这里比较懒就不该了
当数据更新的时候更新对应的add_time 字段 这样就可以检测到该数据为需要更新同步的数据  这里数据可以被检测到了  但是如何实现 同步更新而不是 增量去插入呢

–incremental lastmodified
  基于时间列的增量导入（将时间列大于等于阈值的所有数据增量导入Hadoop）


  –check-column
  时间列（int）


  –last-value
  阈值（int）


  –merge-key
  合并列（主键，合并键值相同的记录）

这种方式可以选取merge-key 该参数是设置需要update的字段 可以理解为主键 若该字段在已有数据存在则更新 没有则插入

到这里问题又来了  我第一次设置last-value 时间阈值 那后续时间一直推进以往的数据都已经同步过了 我还要动态去更改last-value 吗?
这里 网上的各种资料表明 时间会自动更新  经过验证 的确会自动更新last-value 时间阈值 前提是使用job的方式！

如何创建job 命令自行百度  这里需要提醒的是 启动job输入的是数据库密码 不是linux账户密码。。。

到这里解释一下为什么同步的只能是hdfs 不能是hive  因为sqoop目前不支持hive 以后会不会更新还不知道

![sqoop](https://raw.githubusercontent.com/MrDLontheway/mrdlontheway.github.io/master/images/sqoop1.png)


<h3>3.并发控制</h3>
我们知道通过 -m 参数能够设置导入数据的 map 任务数量，即指定了 -m 即表示导入方式为并发导入，这时我们必须同时指定 --split-by 参数指定根据哪一列来实现哈希分片，从而将不同分片的数据分发到不同 map 任务上去跑，避免数据倾斜。

ps:
-  生产环境中，为了防止主库被Sqoop抽崩，我们一般从备库中抽取数据。
-  一般RDBMS的导出速度控制在60~80MB/s，每个 map 任务的处理速度5~10MB/s 估算，即 -m 参数一般设置4~8，表示启动 4~8 个map 任务并发抽取。

这里提一下 踩得坑  因为CDH用的 cloudrea jdk 1.7  而我系统的环境变量为jdk 1.8  会报错 字节码文件不兼容 所以一定要保证执行sqoop job的机器jdk与 cdh jdk一致

目前hive的同步 还没有尝试 但是可以间接同步就是直接sqoop 同步更新到hive 的hdfs元数据目录这样搭到目的 还未尝试

另外一种方式 hive2.2 增加了merge into 的操作 可以需要更新的数据存入hive临时表 然后把 临时表和主表merge into 合并操作 hive 表必须是“事务表”（即：支持事务)

