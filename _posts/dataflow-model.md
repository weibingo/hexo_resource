---
title: Dataflow模型
date: 2018-12-21 20:49:58
tags:
 - 大数据
 - 流式计算
 - spark
 - flink
categories: 
 - 数据工程
 - 流式计算
---
Dataflow模型：是谷歌在处理无边界数据的实践中，总结的一套SDK级别的解决方案，其目标是做到在非有序的，无边界的海量数据上，基于事件时间进行运算，并能根据数据自身的属性进行window操作，同时数据处理过程的正确性，延迟，代价可根据需求进行灵活的调整配置。
### DataFlow模型核心
和Spark通过micro batch模型来处理Streaming场景的出发点不同，Dataflow认为batch的处理模式只是streaming处理模式的一个子集。在无边界数据集的处理过程中，要及时产出数据结果，无限等待显然是不可能的，所以必然需要对要处理的数据划定一个窗口区间，从而对数据及时的进行分段处理和产出，而各种处理模式（stream，micro batch，session，batch），本质上，只是窗口的大小不同，窗口的划分方式不同而已。Batch的处理模式就只是一个窗口区间涵盖了整个有边界的数据集这样的一种特例场景而已。一个设计良好的能处理无边界数据集的系统，完全能在准确性和正确性上做到和“Batch”系统一样甚至应该更好。
<!--more-->
Dataflow模型里强调的两个时间概念：Event time和 Process time：
* Event time 事件时间: 就是数据真正发生的时间，比如用户浏览了一个页面，或者下了一个订单等等，这时候通常就会有一些数据会被生产出来，比如前者可能会产生一条用户的浏览日志
* Process time: 则是这条日志数据真正到达计算框架中被处理的时间点，简单的说，就是你的程序是什么时候读到这条日志的

现实情况下，由于各种原因，数据采集，传输到达处理系统的时间可能会有长短不同的延迟，在分布式应用场景环境下，不仅是延迟，数据乱序到达往往也是常态。这些问题，在有边界数据集的处理过程中往往并不存在，或者无关紧要。

但在无边界数据集中，乱序，延迟问题就显得很关键。在Dataflow模型中，数据的处理过程中需要解决的问题，被概括为以下4个方面：
* What results are being computed ： 计算逻辑是什么
* Where in event time they are being computed ： 计算什么时候（事件时间）的数据
* When in processing time they are materialized ： 在什么时候（处理时间）进行计算/输出
* How earlier results relate to later refinements ： 后续数据如何影响（修正）之前的计算结果

针对以上四个问题，提出了三个模型：
* 一个支持基于事件时间的窗口（window）模型，并提供简易的API接口：支持固定窗口／滑动窗口／Session（以Key为维度，基于事件时间连续性进行划分）等窗口模式。（**解决Where问题**）
* 一个和数据自身特性绑定的计算结果输出触发模型，并提供灵活可描述的API接口。（**解决when问题**）
* 一个增量更新模型，可以将数据增量更新的能力融合进上述窗口和结果触发模型中。（**解决how问题**）
### 三个模型
#### windowing
统计窗口，对于unbounded data，只能基于windowing做处理。windowing有以下三种：
![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/dataflow-model/windowing.png)

### 参考
https://www.cnblogs.com/fxjwind/p/5124174.html
https://www.cnblogs.com/tgzhu/p/7656508.html