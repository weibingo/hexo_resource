---
title: livy指南
date: 2018-12-13 20:49:58
tags:
 - livy
 - spark
categories: 数据工程
---
Livy是一个基于Spark的开源REST服务，它能够通过REST的方式将代码片段或是序列化的二进制代码提交到Spark集群中去执行。它提供了以下这些基本功能：

* 提交Scala、Python或是R代码片段到远端的Spark集群上执行；
* 提交Java、Scala、Python所编写的Spark作业到远端的Spark集群上执行；
* 提交批处理应用在集群中运行。

### livy安装
livy安装很简单，从[官网](http://livy.incubator.apache.org/download/)下载zip包。
解压，设置spark和hadoop的环境。
```bash
export SPARK_HOME=spark的路径
export HADOOP_CONF_DIR=/etc/hadoop/conf （hadoop配置路径）
```

然后在livy的bin目录下执行：
livy-server start

### livy配置
在conf/livy.conf文件中有配置，常用的配置：

| 配置信息 | 含义 |
| ----- | ----- |
|livy.server.host = 0.0.0.0 |livy服务启动的host|
|livy.server.port = 80| 启动端口号|
|livy.spark.master = yarn|livy执行spark master|
|livy.spark.deploy-mode = client|spark启动模式|
|livy.repl.enable-hive-context = true|启动hiveContext|
|livy.ui.enabled = true|livy ui页面|

还有其他的配置，用户可以根据自己需求自行配置。
另外，当集群开启了kerberos验证，执行spark任务时报gss错误，需要在配置中加上：
```
livy.server.launch.kerberos.keytab = /conf/xxx.keytab
livy.server.launch.kerberos.principal = xxx@HADOOP.COM
```
<!-- more -->
### livy的使用
livy提供了两种执行方式。
* 一种是通过rest接口，见[官网](http://livy.incubator.apache.org/docs/latest/rest-api.html)接口定义。
用户可以以REST请求的方式通过Livy启动一个新的Spark集群，Livy将每一个启动的Spark集群称之为一个会话（session），一个会话是由一个完整的Spark集群所构成的，并且通过RPC协议在Spark集群和Livy服务端之间进行通信。根据处理交互方式的不同，Livy将会话分成了两种类型：
  * 交互式会话（interactive session），这与Spark中的交互式处理相同，交互式会话在其启动后可以接收用户所提交的代码片段，在远端的Spark集群上编译并执行；
  * 批处理会话（batch session），用户可以通过Livy以批处理的方式启动Spark应用，这样的一个方式在Livy中称之为批处理会话，这与Spark中的批处理是相同的。
可以看到，Livy所提供的核心功能与原生Spark是相同的，它提供了两种不同的会话类型来代替Spark中两类不同的处理交互方式。接下来我们具体了解一下这两种类型的会话。

接口都是异步提交形式，就是你提交一个batch任务或者解释执行一段code。如果提交是个batch任务，会创建一个batch，返回batchId，然后你通过GET /batches/{batchId}或者执行状态，和执行结果，这也是立即返回。如果想同步获取结果，需要程序自己循环获取状态，当success后再退出循环。解释执行方式类似，创建session kind，然后在POST /sessions/{sessionId}/statements中提交code。通过GET /sessions/{sessionId}/statements/{statementId}获取执行状态，也是异步返回。
batch任务提交示例：
```bash
curl -X POST -H "Content-Type: application/json" -d '{"file":"hdfs://xx/user/laiwb/spark-jars/spark-ml.jar","className":"com.xx.data.livy.SparkExp","conf":{"spark.dynamicAllocation.enabled":"true"}}' http://192.168.18.10:80/batches
```

获取batch任务：
```
curl -X GET  http://192.168.18.10:80/batches/{batchId}
```

* 通过[livy Programmatic](http://livy.incubator.apache.org/docs/latest/programmatic-api.html)形式。livy封装了sparkContext，添加了对象共享功能（setSharedObject）。下面是spark代码与livy代码对比图：

![](http://hexo-1256892004.cos.ap-beijing.myqcloud.com/livy-guide/livy-api.png)

上图是pyspark，livy也提供了java/scala API。spark程序需要继承Job类，覆盖Job类的call()方法。示例如下：
```scala
class SparkExample extends Job[Double]{
  override def call(jobContext: JobContext): Double = {
    val spark = jobContext.hivectx()
    val db = spark.sql("select max(cnt) max, min(cnt) min, avg(cnt) avg ,stddev_pop(cnt) std from laiwb.user_xx where cast(substr(uid,4) as int) > 100000000 ")
    jobContext.setSharedObject("avg", db)
    val dou = db.collect().head.getDouble(2)
    dou
  }
}
```
定义一个spark任务SparkExample，并将其打成jar包，上传到hdfs，也可以放在本地目录中，然后写Submit类。Submit类主要代码如下：
```
// 定义一个livy client 连接，url为livy服务的地址
val client = new LivyClientBuilder(false).setURI(new URI(url)).build()  
// 将SparkExample类打好的jar包（已经上传至hdfs）加入client中。如果jar包在本地，使用client.uploadJar(new File(jar)).get()
client.addJar(new URI(jar))
// main class
val spe = Class.forName("com.xx.data.livy.SparkExample").newInstance.asInstanceOf[SparkExample]
// 提交任务，submit()方法返回的是JobHandle对象，继承于Future，是个异步任务。如果想等待任务执行，可以通过get()方法。
// dd便是livy spark任务执行完返回的double值
val dd = client.submit(spe).get()
```

在SparkExample类中添加了
```
jobContext.setSharedObject("avg", db)
```
这是livy提供的共享对象功能，在其他任务中可以通过getSharedObject()通过key获取到此共享对象。
如果一个任务直接通过getSharedObject()，可以在spark stage界面上看到此对象的task是skip状态，说明它是被缓存起来，不在从DAG的起始开始执行。

### 参考
http://livy.incubator.apache.org

https://blog.csdn.net/imgxr/article/details/80130340