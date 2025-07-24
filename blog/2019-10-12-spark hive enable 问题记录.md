---
slug: spark-hive-enable-issue-record
title: spark hive enable 问题记录
authors: [ledai]
tags: [spark hive]
date: 2019-10-12
---

# 关于spark-submit application 程序内未开启hive enable 程序依然会连接hive问题

<!-- truncate -->


## 测试程序 local 运行

```scala
  def main(args: Array[String]): Unit = {
    val spark = SparkSession
      .builder()
      .master("local")
      .appName("test")
      .enableHiveSupport()
      .getOrCreate()

    spark.sqlContext
    spark.sql(
      """
        |select 1
      """.stripMargin).show()

    Thread.sleep(100000000L)
  }
```



```log
19/10/12 16:47:07 INFO internal.SharedState: loading hive config file: file:/Users/daile/IdeaProjects/sparkmanager/spark-DeltaLake/target/classes/hive-site.xml
19/10/12 16:47:07 INFO internal.SharedState: spark.sql.warehouse.dir is not set, but hive.metastore.warehouse.dir is set. Setting spark.sql.warehouse.dir to the value of hive.metastore.warehouse.dir ('/user/hive/warehouse').
19/10/12 16:47:07 INFO internal.SharedState: Warehouse path is '/user/hive/warehouse'.
```



Resources 内包含hive conf 相关配置文件 加载hive 