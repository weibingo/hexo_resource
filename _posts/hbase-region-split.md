---
title: HBase系列之region-split上
date: 2018-10-10 10:32:29
tags: 
 - HBase
 - 数据
categories: HBase
---
Region自动切分是HBase能够拥有良好扩张性的最重要因素之一，也必然是所有分布式系统追求无限扩展性的一副良药。那么region又是如何自动切分的呢，触发的条件又是什么？

### Region切分触发策略
在最新稳定版（1.2.6）中，HBase已经有多达6种切分触发策略。即RegionSplitPolicy的实现子类共有6个，如下类图：
![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/hbase-region-split/split-class.png)

当然，每种触发策略都有各自的适用场景，用户可以根据业务在表级别选择不同的切分触发策略。常见的切分策略如下图：
![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/hbase-region-split/region.png)

#### ConstantSizeRegionSplitPolicy
0.94版本前默认切分策略。这是最容易理解但也最容易产生误解的切分策略，从字面意思来看，当region大小大于某个阈值（hbase.hregion.max.filesize）之后就会触发切分，实际上并不是这样，真正实现中这个阈值是对于某个store来说的，即一个region中最大store的大小大于设置阈值之后才会触发切分。另外一个大家比较关心的问题是这里所说的store大小是压缩后的文件总大小还是未压缩文件总大小，实际实现中store大小为压缩后的文件大小（采用压缩的场景）。ConstantSizeRegionSplitPolicy相对来来说最容易想到，但是在生产线上这种切分策略却有相当大的弊端：切分策略对于大表和小表没有明显的区分。阈值（hbase.hregion.max.filesize）设置较大对大表比较友好，但是小表就有可能不会触发分裂，极端情况下可能就1个，这对业务来说并不是什么好事。如果设置较小则对小表友好，但一个大表就会在整个集群产生大量的region，这对于集群的管理、资源使用、failover来说都不是一件好事。
<!-- more -->
#### IncreasingToUpperBoundRegionSplitPolicy
0.94版本~2.0版本默认切分策略。这种切分策略微微有些复杂，总体来看和ConstantSizeRegionSplitPolicy思路相同，一个region中最大store大小大于设置阈值就会触发切分。但是这个阈值并不像ConstantSizeRegionSplitPolicy是一个固定的值，而是会在一定条件下不断调整，调整规则和region所属表在当前regionserver上的region个数有关系 ：$regions^3$ $\times$ flush size $\times$ 2,即**Region增加策略的初使化大小(其可由配置控制参数为hbase.increasing.policy.initial.size指定；如果没有配置该参数，由取值MemStore的缓存刷新值大小的两倍，MemStore缓存刷新值默认其值为128M，即此时取值256M）$\times$  当前Table Region数的3次方**，当然阈值并不会无限增大，最大值为用户设置的MaxRegionFileSize。这种切分策略很好的弥补了ConstantSizeRegionSplitPolicy的短板，能够自适应大表和小表。而且在大集群条件下对于很多大表来说表现很优秀，但并不完美，这种策略下很多小表会在大集群中产生大量小region，分散在整个集群中。而且在发生region迁移时也可能会触发region分裂。Region增加策略的初使化大小源码如下：
```java
Configuration conf = getConf();
initialSize = conf.getLong("hbase.increasing.policy.initial.size", -1);
if (initialSize > 0) {
  return;
}
HTableDescriptor desc = region.getTableDesc();
if (desc != null) {
  initialSize = 2 * desc.getMemStoreFlushSize();
}
if (initialSize <= 0) {
      initialSize = 2 * conf.getLong(HConstants.HREGION_MEMSTORE_FLUSH_SIZE,
                           HTableDescriptor.DEFAULT_MEMSTORE_FLUSH_SIZE);
    }
```
确定其拆分控制大小的实现方法如下：
```java
/**
   * @return Region max size or {@code count of regions cubed * 2 * flushsize},
   * which ever is smaller; guard against there being zero regions on this server.
   */
  protected long getSizeToCheck(final int tableRegionsCount) {
    // safety check for 100 to avoid numerical overflow in extreme cases
    return tableRegionsCount == 0 || tableRegionsCount > 100
               ? getDesiredMaxFileSize()
               : Math.min(getDesiredMaxFileSize(),
                          initialSize * tableRegionsCount * tableRegionsCount * tableRegionsCount);
  }
```
要想达到每次拆分大小为10G的标准，则需要经过以下4次拆分：
```
第一次split：1^3 * 256 = 256MB 
第二次split：2^3 * 256 = 2048MB 
第三次split：3^3 * 256 = 6912MB 
第四次split：4^3 * 256 = 16384MB > 10GB，因此取较小的值10GB 
后面每次split的size都是10GB了
```
#### SteppingSplitPolicy
2.0版本默认切分策略。这种切分策略的切分阈值又发生了变化，相比IncreasingToUpperBoundRegionSplitPolicy简单了一些，依然和待分裂region所属表在当前regionserver上的region个数有关系，如果region个数等于1，切分阈值为flush size * 2，否则为MaxRegionFileSize。这种切分策略对于大集群中的大表、小表会比IncreasingToUpperBoundRegionSplitPolicy更加友好，小表不会再产生大量的小region，而是适可而止。SteppingSplitPolicy是IncreasingToUpperBoundRegionSplitPolicy的子类，其总共源码只有几行，如下：
```java
public class SteppingSplitPolicy extends IncreasingToUpperBoundRegionSplitPolicy {
  /**
   * @return flushSize * 2 if there's exactly one region of the table in question
   * found on this regionserver. Otherwise max file size.
   * This allows a table to spread quickly across servers, while avoiding creating
   * too many regions.
   */
  protected long getSizeToCheck(final int tableRegionsCount) {
    return tableRegionsCount == 1  ? this.initialSize : getDesiredMaxFileSize();
  }
}
```
#### KeyPrefixRegionSplitPolicy
根据rowKey的前缀对数据进行分组，以便于将这些数据分到相同的Region中，这里是通过指定rowKey的前多少位作为前缀做为拆分控制参数，其参数控制为通过指定Table的描述参数KeyPrefixRegionSplitPolicy.prefix_length（旧版为prefix_split_key_policy.prefix_length）控制拆分前缀的长度，比如rowKey都是16位的，指定前5位是前缀，那么前5位相同的rowKey在进行region split的时候会分到相同的region中。

获取拆分点的实现源码如下：
```java
@Override
  protected byte[] getSplitPoint() {
    byte[] splitPoint = super.getSplitPoint();
    if (prefixLength > 0 && splitPoint != null && splitPoint.length > 0) {
      // group split keys by a prefix
      return Arrays.copyOf(splitPoint,
          Math.min(prefixLength, splitPoint.length));
    } else {
      return splitPoint;
    }
  }
  ```
另外，还有一些其他分裂策略，比如使用DisableSplitPolicy:可以禁止region发生分裂；而DelimitedKeyPrefixRegionSplitPolicy也是让相同的PrefixKey待在一个region中。与KeyPrefixRegionSplitPolicy不同的是，其是根据RowKey中指定分隔字符做为拆分的，显得更加灵活，如RowKey的值为“userid_eventtype_eventid”，且指定了分隔字符串为下划线"__"，则DelimitedKeyPrefixRegionSplitPolicy将取RowKey值中从左往右且第一个分隔字符串之前的字符做为拆分串。

### 参考资料
https://www.codercto.com/a/26841.html

http://hbasefly.com/2017/08/27/hbase-split/