---
title: 使用Spark做数据迁移
date: 2018-11-22 19:50:18
tags: [Spark, HBase, Kafka]
categories: Technology
---

## 背景

源环境使用五台虚拟机搭建的Hadoop集群，原始数据保存在HBase，计算数据保存在ElasticSearch。由于虚拟机的硬件性能和存储空间有限，随着数据量增加，频繁发生资源警告，因此组建了新集群，新集群从Kafka接收业务数据的同时，需要在不影响系统使用的前提下，将旧集群里的数据导入进来。

## 方案

### 方案一

HBase和ElasticSearch各自都有数据迁移方法，可以分别做数据迁移。

+ 优点：HBase和ElasticSearch各自的迁移工具相对成熟，可以保证数据迁移的准确性和速度性。
+ 缺点：需要熟悉两套工具，很难保证集群正常运作的同时完成数据迁移。

### 方案二

将迁移数据如业务数据一样通过Kafka流入集群，旧集群使用Spark应用读出HBase中的原始数据，并写入Kafka；新集群使用Spark应用从Kafka中读出迁移数据，分别写入HBase和ElasticSearch。

+ 优点：迁移数据与业务数据一样方式进入集群，可以复用业务逻辑代码，只需要编写读取数据部分的逻辑代码。
+ 缺点：复用业务数据通道，需要考虑数据流速对集群的影响，否则可能出现数据丢失，影响现行业务流程；迁移数据再走一遍业务流程，浪费计算资源。

### 方案比较

+ 对HBase和ElasticSearch的数据迁移工具都不熟悉，原系统的开发内容主要集中于Spark应用。
+ 迁移数据进入Kafka后和业务数据一样可以被Spark应用处理，只需要区分好数据流通道，代码逻辑基本一致。
+ 使用Spark应用读取旧集群的HBase数据，可以充分利用集群的分布式并发，数据吞吐量大。

综上，使用方案二执行数据迁移

## 实施

### 数据采集

开发Spark应用读取旧集群的HBase数据，有很多成熟的Spark插件可以使用。

+ 使用[shc](https://github.com/hortonworks-spark/shc)读取HBase数据，该插件可以使用Avro schemas方便的定义数据结构，结合Spark SQL只需要几行代码就可以读取出数据。

  ```
  val complex = s"""MAP<int, struct<varchar:string>>"""
  val schema =
    s"""{"namespace": "example.avro",
       |   "type": "record",      "name": "User",
       |    "fields": [      {"name": "name", "type": "string"},
       |      {"name": "favorite_number",  "type": ["int", "null"]},
       |        {"name": "favorite_color", "type": ["string", "null"]}      ]    }""".stripMargin
  val catalog = s"""{
          |"table":{"namespace":"default", "name":"htable"},
          |"rowkey":"key1:key2",
          |"columns":{
            |"col1":{"cf":"rowkey", "col":"key1", "type":"binary"},
            |"col2":{"cf":"rowkey", "col":"key2", "type":"double"},
            |"col3":{"cf":"cf1", "col":"col1", "avro":"schema1"},
            |"col4":{"cf":"cf1", "col":"col2", "type":"string"},
            |"col5":{"cf":"cf1", "col":"col3", "type":"double",        "sedes":"org.apache.spark.sql.execution.datasources.hbase.DoubleSedes"},
            |"col6":{"cf":"cf1", "col":"col4", "type":"$complex"}
          |}
        |}""".stripMargin
     
  val df = sqlContext.read.options(Map("schema1"->schema, HBaseTableCatalog.tableCatalog->catalog)).format("org.apache.spark.sql.execution.datasources.hbase").load()
  ```

+ 使用[Kafka Sink](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html#writing-the-output-of-batch-queries-to-kafka)向Kafka topic中写入数据。

  ```
  df.select(to_json(struct(col("*"))).as("value"))
    .write
    .format("kafka")
    .option("kafka.bootstrap.servers", "192.168.1.1:9200,192.168.1.2:9200")
    .option("topic", "dataexport")
    .save()
  ```

### 数据存储

使用[Spark Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)实现数据流式处理，官网资料写的很详细，这里略过。

## 问题及对策

### 环境相关

+ 分布式集群通常使用Hadoop构建，而Hadoop的配置过程很是繁琐，推荐使用cloudera的[CDH](https://www.cloudera.com/documentation/enterprise/5-15-x.html)，由于当时使用的CDH版本是5.15，它内部集成的Spark版本是1.6，缺少很多新特性，因此还要额外安装[CDS](https://www.cloudera.com/documentation/spark2/latest/topics/spark2.html)，参考官方文档，安装过程很简单。
+ Spark Structured Streaming开发很简单，但是由于是Spark新增加的特性，仍然存在一些问题，因此推荐使用最新的Spark版本。
+ Spark Structured Streaming的Output Sinks还不支持HBase，常用的Spark HBase插件shc仅支持批量写入，使用shc实现流式写入HBase可以参考该[Pull request](https://github.com/hortonworks-spark/shc/pull/238)。

### Spark内存泄漏

+ 使用Spark on Yarn方式运行Structured Streaming应用，长时间运行后会出现Spark driver内存超出Yarn container限制，而使应用被强制杀死，应用退出信息如下：

  ```
  yarn application -status application_1540000486876_0001
  Diagnostics : Application application_1540000486876_0001 failed 2 times due to AM Container for appattempt_1540000486876_0001_000002 exited with  exitCode: -104
  For more detailed output, check application tracking page:http://localhost:8088/proxy/application_1540000486876_0001/Then, click on links to logs of each attempt.
  Diagnostics: Container [pid=76303,containerID=container_1540000486876_0001_02_000001] is running beyond physical memory limits. Current usage: 1.5 GB of 1.5 GB physical memory used; 3.4 GB of 3.1 GB virtual memory used. Killing container.
  ```

  参考网上[答复](https://issues.apache.org/jira/browse/SPARK-23682)应该在Spark 2.4中修复，目前采用如下对策回避：

  1. 使用Triggers减少Spark查询Kafka Topic的次数

     ```
     import org.apache.spark.sql.streaming.Trigger
     
     // ProcessingTime trigger with two-seconds micro-batch interval
     df.writeStream
       .format("console")
       .trigger(Trigger.ProcessingTime("2 seconds"))
       .start()
     ```

  2. 修改Yarn配置，增加Container的内存大小（待验证）

  3. 使用脚本检测Spark应用是否退出，然后重启该应用。

### Kafka数据积压

+ 使用Spark读取HBase数据向Kafka推送，数据吞吐量非常大，通常Kafka设置的数据保留策略是7天覆盖，高吞吐量下，很可能导致Kafka集群的磁盘空间不足；或者下游没有及时读取数据，数据超期或超大小后覆盖而发生丢失。

  采取对策如下：

  1. 修改Kafka Sink的代码，控制数据采集速度。位置如下，实现略。

     ```
     org/apache/spark/sql/kafka010/KafkaWriteTask.scala
     KafkaWriteTask::execute
     ```

  2. 修改Kafka数据保留策略，避免发生磁盘空间不足（例：10小时）

     ```
     bin/kafka-configs.sh --zookeeper localhost:2181 --alter --entity-type topics --entity-name dataexport --add-config retention.ms=36000000
     ```

  3. 以上策略设置的时间需要根据下游数据处理速度而定。

### 应用向Kafka写入数据时异常退出

+ 当HBase单表的数据量非常大时，使用Spark向Kafka写入数据会出现如下异常信息：

  ```
  java.lang.IllegalStateException: Cannot send after the producer is closed.
  ```

  原因是Spark Kafka Sink使用CacheLoader维护Kafka producer实例，并且默认10分钟内不重新获取该实例就会销毁它，Kafka Sink是基于DataFrame的每个Partition获取一次producer实例。当HBase单表数据量大时，一个Region即映射为一个Partition，Kafka Sink处理一个Region的时间很容易超出10分钟。

  + 解决方法是修改`spark.kafka.producer.cache.timeout`参数，增加实例保持时间：

    ```
    spark2-submit --conf spark.kafka.producer.cache.timeout=60m --master local[4] --class com.test.batchprocessing.Application dataprocessing-1.0.jar
    ```