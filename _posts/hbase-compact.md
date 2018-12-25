---
title: HBase系列之compact
date: 2018-10-09 23:31:06
tags: 
 - HBase
 - 数据
categories: HBase
---

在介绍HBase Compaction之前，我们先来看一下HBase是如何存储和操作数据。
![HBase数据存储](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/hbase-compact/hbase-store.png)

如上图所示，HRegionServer负责打开region，并创建对应的HRegion实例。当HRegion打开之后，它会为每个表的HColumnFamily创建一Store实例，ColumnFamily是用户在创建表时定义好的，ColumnFamily在每个region中和Store实例一一对应。每个Store实例包含一个或者多个StoreFile实例，StoreFile是对实际存储数据文件HFile的轻量级封装。每个Store对应一个MemStore（也就是写内存）。一个HRegionServer共享一个HLog实例。

当我们不停地往HBase中写入数据，也就是往MemStore写入数据，HBase会检查MemStore是否达到了需要刷写到磁盘的阈值（更多关于MemStore刷写的信息，可以参考HBase Reference Guide关于MemStore的介绍）。如果达到刷写的条件，MemStore中的记录就会被刷写到磁盘，形成一个新的StoreFile。可想而知，随着MemStore的不断刷写，会形成越来越多的磁盘文件。然而，对于HBase来说，当每个HStore仅包含一个文件时，才会达到最佳的读效率。因此HBase会通过合并已有的HFile来减少每次读数据的磁盘寻道时间，从而提高读速度，这个文件合并过程就称为Compaction。在这里需要说明的是，显然磁盘IO也是有代价的，如果使用不慎的话，不停地重写数据可能会导致网络和磁盘过载。换句话说，compaction其实就是用当前更高的磁盘IO来换取将来更低的磁盘寻道时间。因此，何时执行compaction，其实是一个相当复杂的决策。

Compaction会从一个region的一个store中选择一些hfile文件进行合并。合并说来原理很简单，先从这些待合并的数据文件中读出KeyValues，再按照由小到大排列后写入一个新的文件中。之后，这个新生成的文件就会取代之前待合并的所有文件对外提供服务。HBase的compaction分为minor和major两种，每次触发compact检查，系统会自动决定执行哪一种compaction（合并）。有三种情况会触发compact检查：

* MemStore被刷写到磁盘；
* 用户执行shell命令compact、major_compact或者调用了相应的API；
* HBase后台线程周期性触发检查。

除非是用户使用shell命令major_compact或者调用了majorCompact() API（这种情况会强制HBase执行major合并），在其他的触发情况下，HBase服务器会首先检查上次运行到现在是否达到一个指定的时限。如果没有达到这个时限，系统会选择执行minor合并，接着检查是否满足minor合并的条件。

major合并中会删除那些被标记为删除的数据、超过TTL（time-to-live）时限的数据，以及超过了版本数量限制的数据，将HStore中所有的HFile重写成一个HFile。如此多的工作量，理所当然地，major合并会耗费更多的资源，合并进行时也会影响HBase的响应时间。在HBase 0.96之前，默认每天对region做一次major compact，现在这个周期被改成了7天。然而，因为major compact可能导致某台server短时间内无法响应客户端的请求，如果无法容忍这种情况的话，可以关闭自动major compact，改成在请求低谷期手动触发这一操作。

Minor Compaction是指选取一些小的、相邻的StoreFile将他们合并成一个更大的StoreFile，在这个过程中不会处理已经Deleted或Expired的Cell。一次Minor Compaction的结果是更少并且更大的StoreFile。
<!-- more -->

### compaction流程
整个Compaction始于特定的触发条件，比如flush操作、周期性地Compaction检查操作等。一旦触发，HBase会将该Compaction交由一个独立的线程处理，该线程首先会从对应store中选择合适的hfile文件进行合并，这一步是整个Compaction的核心，选取文件需要遵循很多条件，比如文件数不能太多、不能太少、文件大小不能太大等等，最理想的情况是，选取那些承载IO负载重、文件小的文件集，实际实现中，HBase提供了多个文件选取算法：RatioBasedCompactionPolicy、ExploringCompactionPolicy和StripeCompactionPolicy等，用户也可以通过特定接口实现自己的Compaction算法；选出待合并的文件后，HBase会根据这些hfile文件总大小挑选对应的线程池处理，最后对这些文件执行具体的合并操作。可以通过下图简单地梳理上述流程：
![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/hbase-compact/compact-flow.png)

### 触发时机
HBase中可以触发compaction的因素有很多，最常见的因素有这么三种：Memstore Flush、后台线程周期性检查、手动触发。

* Memstore Flush: 应该说compaction操作的源头就来自flush操作，memstore flush会产生HFile文件，文件越来越多就需要compact。因此在每次执行完Flush操作之后，都会对当前Store中的文件数进行判断，一旦文件数 > hbase.store.compaction.min ，就会触发compaction。需要说明的是，compaction都是以Store为单位进行的，而在Flush触发条件下，整个Region的所有Store都会执行compact，所以会在短时间内执行多次compaction。

* 后台线程周期性检查：后台线程CompactionChecker定期触发检查是否需要执行compaction，检查周期为：hbase.server.thread.wakefrequency $\times$ hbase.server.compactchecker.interval.multiplier。和flush不同的是，该线程优先检查文件数是否大于hbase.store.compaction.min，一旦大于就会触发compaction。如果不满足，它会接着检查是否满足major compaction条件，简单来说，如果当前store中hfile的最早更新时间早于某个值mcTime，就会触发major compaction，HBase预想通过这种机制定期删除过期数据。上文mcTime是一个浮动值，浮动区间默认为［7-7$\times$0.2，7+7$\times$0.2］，其中7为hbase.hregion.majorcompaction，0.2为hbase.hregion.majorcompaction.jitter，可见默认在7天左右就会执行一次major compaction。用户如果想禁用major compaction，只需要将参数hbase.hregion.majorcompaction设为0。

* 手动触发：一般来讲，手动触发compaction通常是为了执行major compaction，原因有三，其一是因为很多业务担心自动major compaction影响读写性能，因此会选择低峰期手动触发；其二也有可能是用户在执行完alter操作之后希望立刻生效，执行手动触发major compaction；其三是HBase管理员发现硬盘容量不够的情况下手动触发major compaction删除大量过期数据；无论哪种触发动机，一旦手动触发，HBase会不做很多自动化检查，直接执行合并。

### 选择合适HFile合并
选择合适的文件进行合并是整个compaction的核心，因为合并文件的大小以及其当前承载的IO数直接决定了compaction的效果。最理想的情况是，这些文件承载了大量IO请求但是大小很小，这样compaction本身不会消耗太多IO，而且合并完成之后对读的性能会有显著提升。然而现实情况可能大部分都不会是这样，在0.96版本和0.98版本，分别提出了两种选择策略，在充分考虑整体情况的基础上选择最佳方案。无论哪种选择策略，都会首先对该Store中所有HFile进行一一排查，排除不满足条件的部分文件：

* 排除当前正在执行compact的文件及其比这些文件更新的所有文件（SequenceId更大）
* 排除某些过大的单个文件，如果文件大小大于hbase.hzstore.compaction.max.size（默认Long最大值），则被排除，否则会产生大量IO消耗

经过排除的文件称为候选文件，HBase接下来会再判断是否满足major compaction条件，如果满足，就会选择全部文件进行合并。判断条件有下面三条，只要满足其中一条就会执行major compaction：

* 用户强制执行major compaction
* 长时间没有进行compact（CompactionChecker的判断条件2）且候选文件数小于hbase.hstore.compaction.max（默认10）
* Store中含有Reference文件，Reference文件是split region产生的临时文件，只是简单的引用文件，一般必须在compact过程中删除

如果不满足major compaction条件，就必然为minor compaction，HBase主要有两种minor策略：RatioBasedCompactionPolicy和ExploringCompactionPolicy，下面分别进行介绍：
#### RatioBasedCompactionPolicy
从老到新逐一扫描所有候选文件，满足其中条件之一便停止扫描：

（1）当前文件大小 < 比它更新的所有文件大小总和 * ratio，其中ratio是一个可变的比例，在高峰期时ratio为1.2，非高峰期为5，也就是非高峰期允许compact更大的文件。那什么时候是高峰期，什么时候是非高峰期呢？用户可以配置参数hbase.offpeak.start.hour和hbase.offpeak.end.hour来设置高峰期。

（2）当前所剩候选文件数 <= hbase.store.compaction.min（默认为3）

停止扫描后，待合并文件就选择出来了，即为当前扫描文件+比它更新的所有文件
#### ExploringCompactionPolicy
该策略思路基本和RatioBasedCompactionPolicy相同，不同的是，Ratio策略在找到一个合适的文件集合之后就停止扫描了，而Exploring策略会记录下所有合适的文件集合，并在这些文件集合中寻找最优解。最优解可以理解为：待合并文件数最多或者待合并文件数相同的情况下文件大小较小，这样有利于减少compaction带来的IO消耗。Ratio策略是0.94版本的默认策略，而0.96版本之后默认策略就换为了Exploring策略。在[clouder文章](https://blog.cloudera.com/blog/2013/12/what-are-hbase-compactions/)中，作者进行了对比，Ratio策略在节省IO方面会有10%左右的提升。
![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/hbase-compact/exploring.png)
### 挑选合适的线程池
HBase实现中有一个专门的线程CompactSplitThead负责接收compact请求以及split请求，而且为了能够独立处理这些请求，这个线程内部构造了多个线程池：largeCompactions、smallCompactions以及splits等，其中splits线程池负责处理所有的split请求，largeCompactions和smallCompaction负责处理所有的compaction请求，其中前者用来处理大规模compaction，后者处理小规模compaction。这里需要明白三点：

1. 上述设计目的是为了能够将请求独立处理，提供系统的处理性能。

2. 哪些compaction应该分配给largeCompactions处理，哪些应该分配给smallCompactions处理？是不是Major Compaction就应该交给largeCompactions线程池处理？不对。这里有个分配原则：待compact的文件总大小如果大于值throttlePoint（可以通过参数hbase.regionserver.thread.compaction.throttle配置，默认为2.5G），分配给largeCompactions处理，否则分配给smallCompactions处理。

3. largeCompactions线程池和smallCompactions线程池默认都只有一个线程，用户可以通过参数hbase.regionserver.thread.compaction.large和hbase.regionserver.thread.compaction.small进行配置。

### 执行HFile文件合并
上文一方面选出了待合并的HFile集合，一方面也选出来了合适的处理线程，万事俱备，只欠最后真正的合并。合并流程说起来也简单，主要分为如下几步：

1. 分别读出待合并hfile文件的KV，并顺序写到位于./tmp目录下的临时文件中

2. 将临时文件移动到对应region的数据目录

3. 将compaction的输入文件路径和输出文件路径封装为KV写入WAL日志，并打上compaction标记，最后强制执行sync

4. 将对应region数据目录下的compaction输入文件全部删除


上述四个步骤看起来简单，但实际是很严谨的，具有很强的容错性和完美的幂等性：

1. 如果RS在步骤2之前发生异常，本次compaction会被认为失败，如果继续进行同样的compaction，上次异常对接下来的compaction不会有任何影响，也不会对读写有任何影响，唯一的影响就是多了一份多余的数据。

2. 如果RS在步骤2之后、步骤3之前发生异常，同样的，仅仅会多一份冗余数据。

3. 如果在步骤3之后、步骤4之前发生异常，RS在重新打开region之后首先会从WAL中看到标有compaction的日志，因为此时输入文件和输出文件已经持久化到HDFS，因此只需要根据WAL移除掉compaction输入文件即可。

### 参考
https://blog.cloudera.com/blog/2013/12/what-are-hbase-compactions/

http://hbasefly.com/2016/07/13/hbase-compaction-1/

http://www.cnblogs.com/yurunmiao/p/3520066.html