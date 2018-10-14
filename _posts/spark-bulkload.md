---
title: spark加速Bulkload
date: 2018-06-12 20:50:49
tags: 
  - spark
  - hbase
categories: 数据工程
---
本文介绍HBase在大数据量导入时Bulkload的操作过程，以及使用spark加速整个Bulkload。

## 背景
在第一次建立Hbase表的时候，我们可能需要往里面一次性导入大量的初始化数据。我们很自然地想到将数据一条条插入到Hbase中，或者通过MR方式等。但是这些方式不是慢就是在导入的过程的占用Region资源导致效率低下，所以很不适合一次性导入大量数据。使用 Bulk Load 方式由于利用了 HBase 的数据信息是按照特定格式存储在 HDFS 里的这一特性，直接在 HDFS 中生成持久化的 HFile 数据格式文件，然后完成巨量数据快速入库的操作，配合 MapReduce 完成这样的操作，不占用 Region 资源，不会产生巨量的写入 I/O，所以需要较少的 CPU 和网络资源。
## Bulkload实现过程

### 利用MR生成HFile
因为HBase底层存储结构是HFile，而Hbase API为我们提供了生成HFile，我们只需要按照要求，写Mapper，生成特定的(ImmutableBytesWritable, Put)对，或者(ImmutableBytesWritable, KeyValue)即可。
<!--more-->
```java
public class BulkLoadMapper extends Mapper<LongWritable, Text, ImmutableBytesWritable, Put>{
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            String line = value.toString();
            String[] items = line.split("\t");
  
            ImmutableBytesWritable rowKey = new ImmutableBytesWritable(items[0].getBytes());
            Put put = new Put(Bytes.toBytes(items[0]));   //ROWKEY
            put.addColumn("f1".getBytes(), "url".getBytes(), items[1].getBytes());
            put.addColumn("f1".getBytes(), "name".getBytes(), items[2].getBytes());
            
            context.write(rowkey, put);
        }
}
```

Mapper中的KEYIN, VALUEIN根据你的文件格式来指定读取类。

MR驱动程序类：
```java
public class BulkLoadDriver {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
        final String SRC_PATH= "hdfs://input";
        final String DESC_PATH= "hdfs://output";
        Configuration conf = HBaseConfiguration.create();
       
        Job job=Job.getInstance(conf);
        job.setJarByClass(BulkLoadDriver.class);
        job.setMapperClass(BulkLoadMapper.class);
        job.setMapOutputKeyClass(ImmutableBytesWritable.class);
        /** 根据VALUEOUT 类别指定Put 或者 KeyValue
         *框架会自行根据 MapOutputValueClass 来决定是使用 KeyValueSortReducer 还是 PutSortReducer
        **/
        job.setMapOutputValueClass(Put.class);  
        job.setOutputFormatClass(HFileOutputFormat2.class);
        HTable table = new HTable(conf,"user_tag");
        HFileOutputFormat2.configureIncrementalLoad(job,table,table.getRegionLocator());
        FileInputFormat.addInputPath(job,new Path(SRC_PATH));
        FileOutputFormat.setOutputPath(job,new Path(DESC_PATH));
          
        System.exit(job.waitForCompletion(true)?0:1);
    }
}
```
### 通过BlukLoad方式加载HFile文件
生成的HFile还需要通过LoadIncrementalHFiles类的doBulkLoad方法，将HFile加载入hbase的目录下。
```java
Configuration conf = HBaseConfiguration.create();
Connection connection = ConnectionFactory.createConnection(configuration);
LoadIncrementalHFiles loder = new LoadIncrementalHFiles(configuration);
loder.doBulkLoad(new Path("hdfs://output"),new HTable(conf,"table_name"));
```

## 使用spark进行Bulkload
在前面生成HFile的步骤中，使用了MR生成，效率较慢，且不能很好的使用hive特性，只能从底层读取hdfs文件进行解析，对于不同格式的数据文件需要进行不同操作。而通过使用spark on hive可以很好的借用hive特性，屏蔽底层数据文件格式，且使用spark也在执行效率上进行提升。
和MR类似，使用spark也是通过把数据转换成(ImmutableBytesWritable, Put)对，或者(ImmutableBytesWritable, KeyValue)对，示例如下：
```scala
val partitionData = data.rdd.map(s => {
      ((s.get(0).toString, s.get(1).toString, s.getString(2)), s.getString(3))
    }).repartitionAndSortWithinPartitions(new SaltPrefixPartitioner(hashV))
    
val hfile = partitionData.map { s =>
      val uid = s._1._1
      val rowkey = uid
      val immutable = new ImmutableBytesWritable(Bytes.toBytes(rowkey))
      val value = s._2
      val keyValue = new KeyValue(Bytes.toBytes(rowkey), Bytes.toBytes(s._1._2),
        Bytes.toBytes(s._1._3), Utils.transferDateToTs(dt), Bytes.toBytes(value))
      (immutable, keyValue)
    }
```
在示例中，data是个DataFrame，schema为：用户id,列族，列名，value。但是我们在查询hive数据时往往表结构为用户id，列1，列2 .... ，所以需要把其转成hbase存储方式（rowkey，family,qualifier,value）。其中有个很重要的一步：repartitionAndSortWithinPartitions。在MR时，MR框架会进行SortReducer，所以刚好满足了HFile有序的要求，但是spark计算生成HFile的过程中只有map类算子，是无序的，所以需要手工进行排序操作。**在Hbase中，需要按照(rowkey，family,qualifier)顺序进行排序**，因为我在Hbase表进行了手工region划分，而region可正好对应spark partition，所以就很好的利用repartitionAndSortWithinPartitions这个算子进行region划分并对region中的数据进行排序了。SaltPrefixPartitioner是我实现的根据用户id进行hash分区的算法。

然后仍然需要定义个Job,和BulkLoadDriver中一样，指定MapOutputKeyClass，MapOutputValueClass，OutputFormatClass，然后使用HFileOutputFormat2.configureIncrementalLoad(job,table,table.getRegionLocator()) ，再调用rdd的算子saveAsNewAPIHadoopFile。
```java
hfile.saveAsNewAPIHadoopFile(stagingFolder, classOf[ImmutableBytesWritable], classOf[KeyValue], classOf[HFileOutputFormat2], job.getConfiguration)
```
这样就按照HFile格式把数据生成在了stagingFolder目录中，最后还需执行Bulkload步骤。
## spark Bulkload的一些注意事项
### kerberos权限问题
在整个过程中，其实是使用了两套账号，在提交spark的过程中，是spark的账号管理，且会提交给yarn(所以账号权限是客户端的账号权限)；而使用MR的话，整个job是通过程序中的configuration配置的，所以你可以对其进行任何权限配置。在Bulkload过程中，也是使用配程序configuration，也是可随意账号配置。而Bulkload只是个普通hdfs操作，并没有通过yarn（也在调研能否通过yarn模式）。
### 数据snappy压缩问题
有时候我们需要把HFile进行压缩，以减少文件存储，在这里我使用了snappy压缩（snappy在存储和CPU计算上相对其他压缩算法更平衡）。因为snappy压缩依赖snappy.so本地方法库，在进行spark计算生成HFile或Bulkload过程中都可能出现这样的错误：
```
java.lang.RuntimeException: native snappy library not available: this version of libhadoop was built without snappy support
```
如果在spark计算生成HFile过程中报错，这就涉及了spark配置读取的问题（因为我们集群对snappy压缩是没问题的，这肯定是在spark任务提交过程中读取了错误的配置项），后续我会专门讲解spark的配置加载。因为我们spark客户端（docker镜像）是官网spark版本，而集群是使用的CDH部署的，所以在配置上有所不同，以及spark文件路径也有所不同。所以问题也出于此，最终在spark-default.conf加上即可。

而在Bulkload过程中，通过LoadIncrementalHFiles源码我们可以知道，因为会根据region进行加载（groupOrSplitPhase），他会去读取HFile文件的位置偏移，如果当前执行Bulkload客户端的hadoop没有snappy库或者配置错误的话，也就读取HFile失败。
## 后续
HBase 官方和cloudera都提供了hbase-spark模块，里面有HbaseContext类，很好的封装了spark在Hbase上面的使用（包括读取Hbase数据转成rdd，Bulkload等），但cloudera版本的使用的spark依然为1.6，所以在使用过程中需要依赖的包也是1.6，而我们docker环境的spark版本是2.0，会有一些问题（看了下code，竟然引入了spark某个不重要的Logging类，然鹅在2.0这个类被移了位置 囧），后续也没有更新了。
Apache hbase-spark_2.0我看源码好像修复了，不在依赖具体spark中的logging类了，但我们CDH中的hbase版本降低，担心会有其他影响，便还未进行测试。