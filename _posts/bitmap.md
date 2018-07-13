最近在标签存储中，需要根据标签值查询用户id，所以想到在源数据表基础上建立索引。因为标签数据量大，且标签基数相对较少，查询条件往往涉及多标签组合过滤，所以选用了bitmap作为索引。
### bitmap简介
bitmap就是以比特位来存储状态。
![bitmap](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/bitmap/bitmap.png)
### bitmap索引
例如用户数据表
![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/bitmap/user_table.png)
现在要在用户性别和婚姻状态建立bitmap索引。
![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/bitmap/user_index.png)
通过索引值为1，我们可以看出性别男的用户rowid为1和3，然后在查询源用户表，就可以查出性别男的是张三，王五。婚姻状况同理。
下面我们如果要查询：
select * from table where Gender='男' and Marital='未婚'
首先我们找到男索引列和未婚索引列，然后对其取并集。
![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/bitmap/intersection.png)
因为是bit存储，索引取并集就相当简单，只需要进行位运算&即可。所以整个bitmap索引多条件过滤效率很高。
### bitmap的适用范围
因为bitmap的特性，所以bitmap索引适用于：
* 查询为主，更新较少的数据
* 基数较少，重复度较多的列
* and/or可以直接通过位运算快速得到结果。

### RoaringBitmap
未压缩的bitmap占有空间大，所以提出了各种的压缩算法。如图：
![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/bitmap/compress.png)
截取[RoaringBitmap中的话](https://github.com/RoaringBitmap/RoaringBitmap)
>Most alternatives to Roaring are part of a larger family of compressed bitmaps that are run-length-encoded bitmaps. They identify long runs of 1s or 0s and they represent them with a marker word. If you have a local mix of 1s and 0, you use an uncompressed word.

>There are many formats in this family:

>* Oracle's BBC is an obsolete format at this point: though it may provide good compression, it is likely much slower than more recent alternatives due to excessive branching.
* WAH is a patented variation on BBC that provides better performance.
* Concise is a variation on the patented WAH. It some specific instances, it can compress much better than WAH (up to 2x better), but it is generally slower.
* EWAH is both free of patent, and it is faster than all the above. On the downside, it does not compress quite as well. It is faster because it allows some form of "skipping" over uncompressed words. So though none of these formats are great at random access, EWAH is better than the alternatives.

>There is a big problem with these formats however that can hurt you badly in some cases: there is no random access. If you want to check whether a given value is present in the set, you have to start from the beginning and "uncompress" the whole thing. This means that if you want to intersect a big set with a large set, you still have to uncompress the whole big set in the worst case...

从中可知，BBC、WAH、Concise、EWAH都是基于run-length-encoded算法，它们很难处理随机读，而且如果想要对两个bitmap取交集，必须解压整个大集合。
基于这些问题，RoaringBitmap提出了另一种压缩方法。

![RoaringBitmap](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/bitmap/roaringbitmap.png)

图中展示了RoaringBitmap的结构，每个RoaringBitmap中都包含一个RoaringArray，名字叫highLowContainer。 
highLowContainer存储了RoaringBitmap中的全部数据。

``
RoaringArray highLowContainer;
``
![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/bitmap/roaringArray.png)
具体的讲解可以参照[RoaringBitmap数据结构及原理](https://blog.csdn.net/yizishou/article/details/78342499)这篇文章。
RoaringBitmap在设计中结合了Array与Bitmap存储，当数据稀疏时，Array存储会更节省空间（试想[3,10000,200000]这三个数如果用bitmap存储会消耗空间远大于直接存储这三个的数组），当数据稠密时，bitmap存储节省空间。而且还提供了RunContainer利用run-length-encoded算法进行压缩（只有在调用runOptimize()方法才会发生转换，会分别和ArrayContainer、BitmapContainer比较空间占用大小，然后选择是否转换）。