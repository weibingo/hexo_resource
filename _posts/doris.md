---
title: doris介绍
date: 2020-12-30 12:16:36
tags:
  - 数据
  - OLAP
categories:
  - 数据工程
  - OLAP
---

# 1.基本概念

## 1.1Doris(Palo) 简介

**Doris 是一个 MPP 的在线 OLAP 系统，主要整合了 Google Mesa （数据模型），Apache Impala （MPP query engine) 和 ORCFile / Parquet (存储格式，编码和压缩) 的技术。**

Doris 具有以下特点：

* 无外部系统依赖
* 高可靠，高可用，高可扩展
* 同时支持 高并发点查询和高吞吐的 Ad-hoc 查询
* 同时支持 批量导入和近实时 mini-batch 导入
* 兼容 MySQL 协议
* 支持 Rollup Table 和 Rollup Table 的智能查询路由
* 支持多表 Join
* 支持 Schema 在线变更
* 支持存储分级，旧的冷数据用 SATA，新的热数据用 SSD

Doris 的系统架构如下:

**Doris 主要分为 FE 和 BE 两种角色，FE 主要负责查询的编译，分发和元数据管理（基于内存，类似 HDFS NN）；BE 主要负责查询的执行和存储系统。**

![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/doris/20201228001916-r6dgky4-doris-fe.png)
<!-- more -->
## 1.2Doris 数据模型

Doris 的数据模型主要分为 3 类:

* Aggregate
* Uniq
* Duplicate

### Aggregate 模型（聚合模型）

Doris 的聚合模型主要用于固定模式的报表类查询场景，实现原理和Mesa 完全一致。

维度列作为 Key, 指标列作为 Value，存储时会按照 **Key 列进行排序**，相同 Key 的 Value 会按照聚合函数 F(Sum, Min, Max, Replace,HLL)进行聚合。

**示例 1：导入数据聚合**

假设业务有如下数据表模式：

| **ColumnName**  | **Type**    | **AggregationType** | **Comment**                    |
| --------------- | ----------- | ------------------- | ------------------------------ |
| user_id         | LARGEINT    |                     | 用户 id                      |
| date            | DATE        |                     | 数据灌入日期             |
| city            | VARCHAR(20) |                     | 用户所在城市             |
| age             | SMALLINT    |                     | 用户年龄                   |
| sex             | TINYINT     |                     | 用户性别                   |
| last_visit_date | DATETIME    | REPLACE             | 用户最后一次访问时间 |
| cost            | BIGINT      | SUM                 | 用户总消费                |
| max_dwell_time  | INT         | MAX                 | 用户最大停留时间       |
| min_dwell_time  | INT         | MIN                 | 用户最小停留时间       |

如果转换成建表语句则如下（省略建表语句中的 Partition 和 Distribution 信息）


```

CREATE TABLE IF NOT EXISTS example_db.expamle_tbl
(
  `user_id` LARGEINT NOT NULL COMMENT "用户id",
  `date` DATE NOT NULL COMMENT "数据灌入日期时间",
  `city` VARCHAR(20) COMMENT "用户所在城市",
  `age` SMALLINT COMMENT "用户年龄",
  `sex` TINYINT COMMENT "用户性别",
  `last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "用户最后一次访问时间",
  `cost` BIGINT SUM DEFAULT "0" COMMENT "用户总消费",
  `max_dwell_time` INT MAX DEFAULT "0" COMMENT "用户最大停留时间",
  `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "用户最小停留时间",
)
AGGREGATE KEY(`user_id`, `date`, `timestamp`, `city`, `age`, `sex`)
... /* 省略 Partition 和 Distribution 信息 */
；
```


可以看到，这是一个典型的用户信息和访问行为的事实表。
在一般星型模型中，用户信息和访问行为一般分别存放在维度表和事实表中。这里我们为了更加方便的解释 Doris 的数据模型，将两部分信息统一存放在一张表中。

表中的列按照是否设置了 AggregationType，分为 Key (维度列) 和 Value（指标列）。没有设置 AggregationType 的，如 user_id、date、age ... 等称为 **Key**，而设置了 AggregationType 的称为 **Value**。

当我们导入数据时，对于 Key 列相同的行和聚合成一行，而 Value 列会按照设置的 AggregationType 进行聚合。 AggregationType 目前有以下四种聚合方式：

1. SUM：求和，多行的 Value 进行累加。
2. REPLACE：替代，下一批数据中的 Value 会替换之前导入过的行中的 Value。
3. MAX：保留最大值。
4. MIN：保留最小值。

假设我们有以下导入数据（原始数据）：

| **user_id** | **date**   | **city** | **age** | **sex** | **last_visit_date** | **cost** | **max_dwell_time** | **min_dwell_time** |
| ----------- | ---------- | -------- | ------- | ------- | ------------------- | -------- | ------------------ | ------------------ |
| 10000       | 2017-10-01 | 北京   | 20      | 0       | 2017-10-01 06:00:00 | 20       | 10                 | 10                 |
| 10000       | 2017-10-01 | 北京   | 20      | 0       | 2017-10-01 07:00:00 | 15       | 2                  | 2                  |
| 10001       | 2017-10-01 | 北京   | 30      | 1       | 2017-10-01 17:05:45 | 2        | 22                 | 22                 |
| 10002       | 2017-10-02 | 上海   | 20      | 1       | 2017-10-02 12:59:12 | 200      | 5                  | 5                  |
| 10003       | 2017-10-02 | 广州   | 32      | 0       | 2017-10-02 11:20:00 | 30       | 11                 | 11                 |
| 10004       | 2017-10-01 | 深圳   | 35      | 0       | 2017-10-01 10:00:15 | 100      | 3                  | 3                  |
| 10004       | 2017-10-03 | 深圳   | 35      | 0       | 2017-10-03 10:20:22 | 11       | 6                  | 6                  |

我们假设这是一张记录用户访问某商品页面行为的表。我们以第一行数据为例，解释如下：

| **数据**          | **说明**                                                |
| ------------------- | --------------------------------------------------------- |
| 10000               | 用户 id，每个用户唯一识别 id                   |
| 2017-10-01          | 数据入库时间，精确到日期                      |
| 北京              | 用户所在城市                                        |
| 20                  | 用户年龄                                              |
| 0                   | 性别男（1 代表女性）                             |
| 2017-10-01 06:00:00 | 用户本次访问该页面的时间，精确到秒       |
| 20                  | 用户本次访问产生的消费                         |
| 10                  | 用户本次访问，驻留该页面的时间             |
| 10                  | 用户本次访问，驻留该页面的时间（冗余） |

那么当这批数据正确导入到 Doris 中后，Doris 中最终存储如下：

| **user_id** | **date**   | **city** | **age** | **sex** | **last_visit_date** | **cost** | **max_dwell_time** | **min_dwell_time** |
| ----------- | ---------- | -------- | ------- | ------- | ------------------- | -------- | ------------------ | ------------------ |
| 10000       | 2017-10-01 | 北京   | 20      | 0       | 2017-10-01 07:00:00 | 35       | 10                 | 2                  |
| 10001       | 2017-10-01 | 北京   | 30      | 1       | 2017-10-01 17:05:45 | 2        | 22                 | 22                 |
| 10002       | 2017-10-02 | 上海   | 20      | 1       | 2017-10-02 12:59:12 | 200      | 5                  | 5                  |
| 10003       | 2017-10-02 | 广州   | 32      | 0       | 2017-10-02 11:20:00 | 30       | 11                 | 11                 |
| 10004       | 2017-10-01 | 深圳   | 35      | 0       | 2017-10-01 10:00:15 | 100      | 3                  | 3                  |
| 10004       | 2017-10-03 | 深圳   | 35      | 0       | 2017-10-03 10:20:22 | 11       | 6                  | 6                  |

可以看到，用户 10000 只剩下了一行**聚合后**的数据。而其余用户的数据和原始数据保持一致。这里先解释下用户 10000 聚合后的数据：

前 5 列没有变化，从第 6 列 last_visit_date 开始：

* 2017-10-01 07:00:00：因为 last_visit_date 列的聚合方式为 REPLACE，所以 2017-10-01 07:00:00 替换了 2017-10-01 06:00:00 保存了下来。
  > 注：在同一个导入批次中的数据，对于 REPLACE 这种聚合方式，替换顺序不做保证。如在这个例子中，最终保存下来的，也有可能是 2017-10-01 06:00:00。而对于不同导入批次中的数据，可以保证，后一批次的数据会替换前一批次。
  >
* 35：因为 cost 列的聚合类型为 SUM，所以由 20 + 15 累加获得 35。
* 10：因为 max_dwell_time 列的聚合类型为 MAX，所以 10 和 2 取最大值，获得 10。
* 2：因为 min_dwell_time 列的聚合类型为 MIN，所以 10 和 2 取最小值，获得 2。

经过聚合，Doris 中最终只会存储聚合后的数据。换句话说，即明细数据会丢失，用户不能够再查询到聚合前的明细数据了。

### **Uniq 模型（唯一主键）**

在某些多维分析场景下，用户更关注的是如何保证 Key 的唯一性，即如何获得 Primary Key 唯一性约束。因此，我们引入了 Uniq 的数据模型。该模型本质上是聚合模型的一个特例，也是一种简化的表结构表示方式。我们举例说明。

| **ColumnName** | **Type**     | **IsKey** | **Comment**        |
| -------------- | ------------ | --------- | ------------------ |
| user_id        | BIGINT       | Yes       | 用户 id          |
| username       | VARCHAR(50)  | Yes       | 用户昵称       |
| city           | VARCHAR(20)  | No        | 用户所在城市 |
| age            | SMALLINT     | No        | 用户年龄       |
| sex            | TINYINT      | No        | 用户性别       |
| phone          | LARGEINT     | No        | 用户电话       |
| address        | VARCHAR(500) | No        | 用户住址       |
| register_time  | DATETIME     | No        | 用户注册时间 |

这是一个典型的用户基础信息表。这类数据没有聚合需求，只需保证主键唯一性。（这里的主键为 user_id + username）。那么我们的建表语句如下：


```
CREATE TABLE IF NOT EXISTS example_db.expamle_tbl(
  `user_id` LARGEINT NOT NULL COMMENT "用户id",
  `username` VARCHAR(50) NOT NULL COMMENT "用户昵称",
  `city` VARCHAR(20) COMMENT "用户所在城市",
  `age` SMALLINT COMMENT "用户年龄",
  `sex` TINYINT COMMENT "用户性别",
  `phone` LARGEINT COMMENT "用户电话",
  `address` VARCHAR(500) COMMENT "用户地址",
  `register_time` DATETIME COMMENT "用户注册时间"
) UNIQUE KEY(`user_id`, `user_name`)
... /* 省略 Partition 和 Distribution 信息 */
；
```

而这个表结构，完全同等于以下使用聚合模型描述的表结构：

| **ColumnName** | **Type**     | **AggregationType** | **Comment**        |
| -------------- | ------------ | ------------------- | ------------------ |
| user_id        | BIGINT       |                     | 用户 id          |
| username       | VARCHAR(50)  |                     | 用户昵称       |
| city           | VARCHAR(20)  | REPLACE             | 用户所在城市 |
| age            | SMALLINT     | REPLACE             | 用户年龄       |
| sex            | TINYINT      | REPLACE             | 用户性别       |
| phone          | LARGEINT     | REPLACE             | 用户电话       |
| address        | VARCHAR(500) | REPLACE             | 用户住址       |
| register_time  | DATETIME     | REPLACE             | 用户注册时间 |

及建表语句：

```
CREATE TABLE IF NOT EXISTS example_db.expamle_tbl(
  `user_id` LARGEINT NOT NULL COMMENT "用户id",
  `username` VARCHAR(50) NOT NULL COMMENT "用户昵称",
  `city` VARCHAR(20) REPLACE COMMENT "用户所在城市",
  `age` SMALLINT REPLACE COMMENT "用户年龄",
  `sex` TINYINT REPLACE COMMENT "用户性别",
  `phone` LARGEINT REPLACE COMMENT "用户电话",
  `address` VARCHAR(500) REPLACE COMMENT "用户地址",
  `register_time` DATETIME REPLACE COMMENT "用户注册时间"
)
AGGREGATE KEY(`user_id`, `user_name`)
... /* 省略 Partition 和 Distribution 信息 */
；
```

即 Uniq 模型完全可以用聚合模型中的 REPLACE 方式替代。其内部的实现方式和数据存储方式也完全一样。这里不再继续举例说明。

### Duplicate 模型（冗余模型）

由于聚合模型存在下面的缺陷，Doris 引入了非聚合模型。

* 必须区分维度列和指标列
* 维度列很多时，Sort 的成本很高。
* Count 成本很高，需要读取所有维度列（可以参考 Kylin 的解决方法进行优化）

非聚合模型主要用于Ad-hoc 查询，不会有任何聚合，不区分维度列和指标列，但是在建表时**需要指定 Sort Columns**，**数据导入时会根据 Sort Columns 进行排序**，查询时根据 Sort Column 过滤会比较高效。

![](https://km.sankuai.com/api/file/387029494/387077330)

| **ColumnName** | **Type**      | **SortKey** | **Comment**        |
| -------------- | ------------- | ----------- | ------------------ |
| timestamp      | DATETIME      | Yes         | 日志时间       |
| type           | INT           | Yes         | 日志类型       |
| error_code     | INT           | Yes         | 错误码          |
| error_msg      | VARCHAR(1024) | No          | 错误详细信息 |
| op_id          | BIGINT        | No          | 负责人 id       |
| op_time        | DATETIME      | No          | 处理时间       |

建表语句如下：


```
CREATE TABLE IF NOT EXISTS example_db.expamle_tbl
(
  `timestamp` DATETIME NOT NULL COMMENT "日志时间",
  `type` INT NOT NULL COMMENT "日志类型",
  `error_code` INT COMMENT "错误码",
  `error_msg` VARCHAR(1024) COMMENT "错误详细信息",
  `op_id` BIGINT COMMENT "负责人id",
  `op_time` DATETIME COMMENT "处理时间"
)
DUPLICATE KEY(`timestamp`, `type`)
... /* 省略 Partition 和 Distribution 信息 */
；
```


### ROLLUP

ROLLUP 在多维分析中是“上卷”的意思，即将数据按某种指定的粒度进行进一步聚合。

基本概念

在 Doris 中，我们将用户通过建表语句创建出来的表成为 Base 表（Base Table）。Base 表中保存着按用户建表语句指定的方式存储的基础数据。

在 Base 表（同一个分区内）之上，我们可以创建任意多个 ROLLUP 表。这些 ROLLUP 的数据是基于 Base 表产生的，并且在物理上是**独立存储**的。

ROLLUP 表的基本作用，在于在 Base 表的基础上，获得更粗粒度的聚合数据。

在 Kylin 中，我们把每一种维度组合称之为 Cuboid,在 Doris 中与之等价的概念是 RollUp 表。实际上，**Kylin 的 Cuboid 和 Doris 的 RollUp 表都可以认为是一种 Materialized Views 或者 Index。**

Doris 的 RollUp 表 和 Kylin 的 Cuboid 一样，**在查询时不需要显示指定**，系统内部会根据查询条件进行智能路由。下图是个 RollUp 表的示意。

![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/doris/20201228001950-0tz07wg-rollup.png)

**Doris RollUp 表的路由规则**如下：

1. 选择包含所有查询列的 RollUp 表
2. 按照过滤和排序的 column 筛选最符合的 RollUp 表
3. 按照 Join 的 column 筛选最符合的 RollUp 表
4. 行数最小的
5. 列数最小的

|                    | Doris RollUp                                                            | Kylin Cuboid                                                                                              |
| ------------------ | ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| 定义的成本    | 需要手动逐个定义                                                | 系统根据 Web 上维度，聚集组的设置自动定义出所有 Cuboid                               |
| 定义的灵活性 | 维度列和指标列可以自由选择                                 | 只可以选择维度列，每个 Cuboid 都必须包含所有指标列                                   |
| 计算方式       | 从原始数据直接生成每个 RollUp 表的数据                   | 根据 Cuboid Tree 分层构建 Cuboid，每个 Cuboid 的输入是 Parent cuboid，不是原始数据。 |
| 物理存储       | 每个 RollUp 表是独立存储的                                     | 多个 Cuboid 会存储到 1 个 HFile 中(按照大小)                                                  |
| 查询路由       | 会根据过滤列，排序列，Join 列，行数，列数进行路由 | 仅会根据维度列进行路由                                                                         |

下面我们用示例详细说明在不同数据模型中的 ROLLUP 表及其作用。

示例 1：获得每个用户的总消费

接 **Aggregate 模型**小节的**示例 2**，Base 表结构如下：

| **ColumnName**  | **Type**    | **AggregationType** | **Comment**                       |
| --------------- | ----------- | ------------------- | --------------------------------- |
| user_id         | LARGEINT    |                     | 用户 id                         |
| date            | DATE        |                     | 数据灌入日期                |
| timestamp       | DATETIME    |                     | 数据灌入时间，精确到秒 |
| city            | VARCHAR(20) |                     | 用户所在城市                |
| age             | SMALLINT    |                     | 用户年龄                      |
| sex             | TINYINT     |                     | 用户性别                      |
| last_visit_date | DATETIME    | REPLACE             | 用户最后一次访问时间    |
| cost            | BIGINT      | SUM                 | 用户总消费                   |
| max_dwell_time  | INT         | MAX                 | 用户最大停留时间          |
| min_dwell_time  | INT         | MIN                 | 用户最小停留时间          |

存储的数据如下：

| **user_id** | **date**   | **timestamp**       | **city** | **age** | **sex** | **last_visit_date** | **cost** | **max_dwell_time** | **min_dwell_time** |
| ----------- | ---------- | ------------------- | -------- | ------- | ------- | ------------------- | -------- | ------------------ | ------------------ |
| 10000       | 2017-10-01 | 2017-10-01 08:00:05 | 北京   | 20      | 0       | 2017-10-01 06:00:00 | 20       | 10                 | 10                 |
| 10000       | 2017-10-01 | 2017-10-01 09:00:05 | 北京   | 20      | 0       | 2017-10-01 07:00:00 | 15       | 2                  | 2                  |
| 10001       | 2017-10-01 | 2017-10-01 18:12:10 | 北京   | 30      | 1       | 2017-10-01 17:05:45 | 2        | 22                 | 22                 |
| 10002       | 2017-10-02 | 2017-10-02 13:10:00 | 上海   | 20      | 1       | 2017-10-02 12:59:12 | 200      | 5                  | 5                  |
| 10003       | 2017-10-02 | 2017-10-02 13:15:00 | 广州   | 32      | 0       | 2017-10-02 11:20:00 | 30       | 11                 | 11                 |
| 10004       | 2017-10-01 | 2017-10-01 12:12:48 | 深圳   | 35      | 0       | 2017-10-01 10:00:15 | 100      | 3                  | 3                  |
| 10004       | 2017-10-03 | 2017-10-03 12:38:20 | 深圳   | 35      | 0       | 2017-10-03 10:20:22 | 11       | 6                  | 6                  |

在此基础上，我们创建一个 ROLLUP：

| **ColumnName** |
| -------------- |
| user_id        |
| cost           |

该 ROLLUP 只包含两列：user_id 和 cost。则创建完成后，该 ROLLUP 中存储的数据如下：

| **user_id** | **cost** |
| ----------- | -------- |
| 10000       | 35       |
| 10001       | 2        |
| 10002       | 200      |
| 10003       | 30       |
| 10004       | 111      |

可以看到，ROLLUP 中仅保留了每个 user_id，在 cost 列上的 SUM 的结果。那么当我们进行如下查询时:

SELECT user_id, sum(cost) FROM table GROUP BY user_id;

Doris 会自动命中这个 ROLLUP 表，从而只需扫描极少的数据量，即可完成这次聚合查询

### 多版本

为了获得更高的导入吞吐量，Doris 的数据更新是按照 batch 来更新的。为了在**数据更新时不影响数据查询**以及**保证更新的原子性**，Doris 采用了 **MVCC** 的方式，所以在数据更新时每个 batch 都需要指定一个 verison。

数据的版本化虽然可以解决读写冲突和更新的原子性，但是也带来了以下问题：

1. **存储成本**。 多版本意味着我们需要存储多份数据，但是由于聚合后的数据一般比较小，所以这个问题还好。
2. **查询时延**。 如果有很多版本，那么查询时需要遍历的版本数据就会很多，查询时延自然就会增大。

为了解决这两个问题，常见的思路就是及时删除不需要的、过期的数据，以及将小的文件 Merge 为大的文件。

![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/doris/20201228002026-7l9ng8w-merge.png)

![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/doris/20201228002106-csq5fmi-delta-compaction.png)

如上图所示，Mesa 的 Merge 策略和 HBase 很像。

类似 HBase 的 minor compaction 和 major compaction，Mesa 中引入了***cumulative compaction***和***base compaction***的概念。

Mesa 中将包含了一定版本的数据称为***deltas***, 表示为[V1, V2]，刚实时写入的小 deltas， 称之为***singleton deltas***，然后每到一定的版本数(图中是 10)，就通过 cumulative compaction 将 10 个 singleton deltas 合并为 1 个 cumulative deltas，最终每天会通过 base compaction 将一定周期内所有的 deltas 都合并为***base deltas***。

所以查询时一般只需要查询 1 个 base deltas， 1 个 cumulative deltas 和少数 singleton deltas 即可。

注意，compaction 是在**后台并发和异步执行**的，此外由于 Mesa 的存储是按照 key 有序存储的，所以 deltas 的 merge 是**线性时间**的。

### 前缀索引

不同于传统的数据库设计，Doris 不支持在任意列上创建索引。Doris 这类 MPP 架构的 OLAP 数据库，通常都是通过提高并发，来处理大量数据的。
本质上，Doris 的数据存储在类似 SSTable（Sorted String Table）的数据结构中。该结构是一种有序的数据结构，可以按照指定的列进行排序存储。在这种数据结构上，以排序列作为条件进行查找，会非常的高效。

在 Aggregate、Uniq 和 Duplicate 三种数据模型中。底层的数据存储，是按照各自建表语句中，AGGREGATE KEY、UNIQ KEY 和 DUPLICATE KEY 中指定的列进行排序存储的。

而前缀索引，即在排序的基础上，实现的一种根据给定前缀列，快速查询数据的索引方式。

我们将一行数据的前 **36 个字节** 作为这行数据的前缀索引。当遇到 VARCHAR 类型时，前缀索引会直接截断。我们举例说明：

1. 以下表结构的前缀索引为 user_id(8Byte) + age(8Bytes) + message(prefix 20 Bytes)。

| **ColumnName** | **Type**     |
   | ---------------- | -------------- |
   | user_id        | BIGINT       |
   | age            | INT          |
   | message        | VARCHAR(100) |
   | max_dwell_time | DATETIME     |
   | min_dwell_time | DATETIME     |
2. 以下表结构的前缀索引为 user_name(20 Bytes)。即使没有达到 36 个字节，因为遇到VARCHAR，所以直接截断，不再往后继续。

| **ColumnName** | **Type**     |
   | ---------------- | -------------- |
   | user_name      | VARCHAR(20)  |
   | age            | INT          |
   | message        | VARCHAR(100) |
   | max_dwell_time | DATETIME     |
   | min_dwell_time | DATETIME     |

当我们的查询条件，是**前缀索引的前缀**时，可以极大的加快查询速度。比如在第一个例子中，我们执行如下查询：

SELECT * FROM table WHERE user_id=1829239 and age=20；

该查询的效率会**远高于**如下查询：

SELECT * FROM table WHERE age=20；

所以在建表时，**正确的选择列顺序，能够极大地提高查询效率**。

ROLLUP 调整前缀索引

因为建表时已经指定了列顺序，所以一个表只有一种前缀索引。这对于使用其他不能命中前缀索引的列作为条件进行的查询来说，效率上可能无法满足需求。因此，我们可以通过创建 ROLLUP 来人为的调整列顺序。举例说明。

Base 表结构如下：

| **ColumnName** | **Type**     |
| -------------- | ------------ |
| user_id        | BIGINT       |
| age            | INT          |
| message        | VARCHAR(100) |
| max_dwell_time | DATETIME     |
| min_dwell_time | DATETIME     |

我们可以在此基础上创建一个 ROLLUP 表：

| **ColumnName** | **Type**     |
| -------------- | ------------ |
| age            | INT          |
| user_id        | BIGINT       |
| message        | VARCHAR(100) |
| max_dwell_time | DATETIME     |
| min_dwell_time | DATETIME     |

可以看到，ROLLUP 和 Base 表的列完全一样，只是将 user_id 和 age 的顺序调换了。那么当我们进行如下查询时：

SELECT * FROM table where age=20 and massage LIKE &quot;%error%&quot;;

会优先选择 ROLLUP 表，因为 ROLLUP 的前缀索引匹配度更高。

### 聚合模型的局限性

这里我们针对 Aggregate 模型（包括 Uniq 模型），来介绍下聚合模型的局限性。

在聚合模型中，模型对外展现的，是**最终聚合后的**数据。也就是说，任何还未聚合的数据（比如说两个不同导入批次的数据），必须通过某种方式，以保证对外展示的一致性。我们举例说明。

假设表结构如下：

| **ColumnName** | **Type** | **AggregationType** | **Comment**        |
| -------------- | -------- | ------------------- | ------------------ |
| user_id        | LARGEINT |                     | 用户 id          |
| date           | DATE     |                     | 数据灌入日期 |
| cost           | BIGINT   | SUM                 | 用户总消费    |

假设存储引擎中有如下两个已经导入完成的批次的数据：

**batch 1**

| **user_id** | **date**   | **cost** |
| ----------- | ---------- | -------- |
| 10001       | 2017-11-20 | 50       |
| 10002       | 2017-11-21 | 39       |

**batch 2**

| **user_id** | **date**   | **cost** |
| ----------- | ---------- | -------- |
| 10001       | 2017-11-20 | 1        |
| 10001       | 2017-11-21 | 5        |
| 10003       | 2017-11-22 | 22       |

可以看到，用户 10001 分属在两个导入批次中的数据还没有聚合。但是为了保证用户只能查询到如下最终聚合后的数据：

| **user_id** | **date**   | **cost** |
| ----------- | ---------- | -------- |
| 10001       | 2017-11-20 | 51       |
| 10001       | 2017-11-21 | 5        |
| 10002       | 2017-11-21 | 39       |
| 10003       | 2017-11-22 | 22       |

我们在查询引擎中加入了聚合算子，来保证数据对外的一致性。

另外，在聚合列（Value）上，执行与聚合类型不一致的聚合类查询时，要注意语意。比如我们在如上示例中执行如下查询：

SELECT MIN(cost) FROM table;

得到的结果是 5，而不是 1。

同时，这种一致性保证，在某些查询中，会极大的降低查询效率。

我们以最基本的 count(*) 查询为例：

SELECT COUNT(*) FROM table;

在其他数据库中，这类查询都会很快的返回结果。因为在实现上，我们可以通过如“导入时对行进行计数，保存 count 的统计信息”，或者在查询时“仅扫描某一列数据，获得 count 值”的方式，只需很小的开销，即可获得查询结果。但是在 Doris 的聚合模型中，这种查询的开销**非常大**。

我们以刚才的数据为例：

**batch 1**

| **user_id** | **date**   | **cost** |
| ----------- | ---------- | -------- |
| 10001       | 2017-11-20 | 50       |
| 10002       | 2017-11-21 | 39       |

**batch 2**

| **user_id** | **date**   | **cost** |
| ----------- | ---------- | -------- |
| 10001       | 2017-11-20 | 1        |
| 10001       | 2017-11-21 | 5        |
| 10003       | 2017-11-22 | 22       |

因为最终的聚合结果为：

| **user_id** | **date**   | **cost** |
| ----------- | ---------- | -------- |
| 10001       | 2017-11-20 | 51       |
| 10001       | 2017-11-21 | 5        |
| 10002       | 2017-11-21 | 39       |
| 10003       | 2017-11-22 | 22       |

所以，select count(*) from table; 的正确结果应该为 **4**。但如果我们只扫描 user_id 这一列，如果加上查询时聚合，最终得到的结果是 **3**（10001, 10002, 10003）。而如果不加查询时聚合，则得到的结果是 **5**（两批次一共 5 行数据）。可见这两个结果都是不对的。

为了得到正确的结果，我们必须同时读取 user_id 和 date 这两列的数据，**再加上查询时聚合**，才能返回 **4** 这个正确的结果。也就是说，在 count(*) 查询中，Doris 必须扫描所有的 AGGREGATE KEY 列（这里就是 user_id 和 date），并且聚合后，才能得到语意正确的结果。当聚合列非常多时，count(*) 查询需要扫描大量的数据。

因此，当业务上有频繁的 count(*) 查询时，我们建议用户通过增加一个**值衡为 1 的，聚合类型为 SUM 的列来模拟 count(*)**。如刚才的例子中的表结构，我们修改如下：

| **ColumnName** | **Type** | **AggreateType** | **Comment**        |
| -------------- | -------- | ---------------- | ------------------ |
| user_id        | BIGINT   |                  | 用户 id          |
| date           | DATE     |                  | 数据灌入日期 |
| cost           | BIGINT   | SUM              | 用户总消费    |
| count          | BIGINT   | SUM              | 用于计算 count |

增加一个 count 列，并且导入数据中，该列值**衡为 1**。则 select count(*) from table; 的结果等价于 select sum(count) from table;。而后者的查询效率将远高于前者。不过这种方式也有使用限制，就是用户需要自行保证，不会重复导入 AGGREGATE KEY 列都相同的行。否则，select sum(count) from table; 只能表述原始导入的行数，而不是 select count(*) from table; 的语义。

另一种方式，就是 **将如上的 count 列的聚合类型改为 REPLACE，且依然值衡为 1**。那么 select sum(count) from table; 和 select count(*) from table; 的结果将是一致的。并且这种方式，没有导入重复行的限制。

### Duplicate 模型

Duplicate 模型没有聚合模型的这个局限性。因为该模型不涉及聚合语意，在做 count(*) 查询时，任意选择一列查询，即可得到语意正确的结果。

### 数据模型的选择建议

因为数据模型在建表时就已经确定，且**无法修改**。所以，选择一个合适的数据模型**非常重要**。

1. Aggregate 模型可以通过预聚合，极大地降低聚合查询时所需扫描的数据量和查询的计算量，非常适合有固定模式的报表类查询场景。但是该模型对 count(*) 查询很不友好。同时因为固定了 Value 列上的聚合方式，在进行其他类型的聚合查询时，需要考虑语意正确性。
2. Uniq 模型针对需要唯一主键约束的场景，可以保证主键唯一性约束。但是无法利用 ROLLUP 等预聚合带来的查询优势（因为本质是 REPLACE，没有 SUM 这种聚合方式）。
3. Duplicate 适合任意维度的 Ad-hoc 查询。虽然同样无法利用预聚合的特性，但是不受聚合模型的约束，可以发挥列存模型的优势（只读取相关列，而不需要读取所有 Key 列）

## 1.3Doris 存储模型

Doris 的存储模型主要整合了 Meda 的数据模型和 ORCFile / Parquet 的存储格式，编码和压缩。

**Doris 存储相关的基本概念**

Doris 元数据上的逻辑概念有 Table，Partition，Tablet，Replica。

Doris 的 Table 支持二级分区，可以先按照日期列进行一级分区，再按照指定列进行 Hash 分桶。

首先 1 个 Table 可以按照日期列分为多个 Partition， 每个 Partition 可以包含多个 Tablet，每个 Table 的数据被水平划分为多个 Tablet，

每个 Tablet 包含若干数据行，***Tablet 是数据移动、复制等操作的最小物理存储单元***，各个 Tablet 之间的数据没有交集，并且在物理上是独立存储的。

***Partition 可以视为逻辑上最小的管理单元，数据的导入与删除，仅能针对一个 Partition 进行***。

1 个 Table 的 Tablet 数量= Partition num * Bucket num。

Tablet 会按照一定大小（**256M**）拆分为多个 segment 文件， segment 是列存的，但是会按行（**1024 行，可配置**）拆分为多个 rowblock。

![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/doris/20201228002138-akghkdf-storage-file.png)

**Doris 的数据文件**

Doris 的数据文件如下图所示：

![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/doris/20201228002150-eolrbk7-file.png)

Doris 数据文件 Stream 的例子：

![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/doris/20201228002202-jypfdh1-stream.png)

**前缀索引**

本质上，Doris 的数据存储是类似 SSTable（Sorted String Table）的数据结构。该结构是一种有序的数据结构，可以按照指定的列进行排序存储。

在这种数据结构上，以排序列作为条件进行查找，会非常的高效。而前缀索引，即在排序的基础上，实现的一种根据给定前缀列，快速查询数据的索引方式。

前缀索引文件的格式如下图所示，索引的 Key 是**每个 rowblock 第一行记录的 Sort Key 的前 36 个字节**，Value 是 **rowblock 在 segment 文件的偏移量**。

有了前缀索引后，我们查询特定 key 的过程就是**两次二分查找**：

1. 先加载 index 文件，二分查找 index 文件获取包含特定 key 的 row blocks 的 offest,然后从 data files 中获取指定的 row blocks；
2. 在 row blocks 中二分查询特定的 key

Index 文件：

![](https://hexo-1256892004.cos.ap-beijing.myqcloud.com/doris/20201228002214-j9xyxca-index.png)

**Min，Max 索引和 Bloomfilter**

在利用前缀索引过滤 block 之前， Doris 也会根据 Min,Max 索引和 bloomfilter（可选）过滤掉不匹配的 block。

**编码和压缩**

**编码**

Doris 中整形的编码方式：（以下几种编码方式的细节具体可以参考 [HIve ORC](https://orc.apache.org/docs/run-length.html)）

1. SHORT_REPEAT
2. DIRECT
3. PATCHED_BASE
4. DELTA

具体选择哪种编码方式会根据数据特点进行选择。

String 会使用字典编码 和 DIRECT 编码，使用哪种方式取决于列的基数。

**压缩**

索引文件和 BF 不会压缩。

数据文件会使用 LZO 或者 LZ4 算法压缩。

**Doris 针对网络传输，硬盘数据，存储有不同的压缩算法**：

* 网络传输时会使用 LZO1X 算法，该算法压缩率低，CPU 开销低
* 硬盘数据会使用 LZO1C_99 算法，该算法压缩率高，CPU 开销大
* 储存会使用 LZ4 算法，压缩率低，CPU 开销低