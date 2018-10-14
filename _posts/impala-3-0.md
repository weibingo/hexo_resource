---
title: Performance optimizations in Apache Impala
tags: impala
categories: 数据工程
---
### 历史和动机
#### SQL on Apache Hadoop
* SQL
* Run on top of HDFS
* Supported various file formats
* Converted query operators to map-reduce jobs
* Run at scale
* Fault-tolerant
* High startup-costs/materialization overhead (slow...)

### impala performance optimizations
#### query planning
2-phase cost-based optimizer
* Phase 1: Generate a single node plan (transformations, join ordering, static partition pruning, runtime filters)
* Phase 2: Convert the single node plan into a distributed query execution plan (add exchange nodes, decide join strategy)
Query fragments (units of work):
* Parts of query execution tree
* Each fragment is executed in one or more impalads

<!-- More -->

#### metadata & statistics

Metadata
• Table metadata (HMS) and block level (HDFS) information are cached to speed-up query time
• Cached data is stored in FlatBuffers to save space and avoid excessive GC
• Metadata loading from HDFS/S3/ADLS uses a thread pool to speedup the operation when needed
Statistics
• Impala uses an HLL to compute Number of distinct values (NDV)
• HLL is much faster than the combination of COUNT and DISTINCT, and uses a constant amount of memory and thus is less memory-intensive for columns with high cardinality.
• HLL size is 1KB per column
• A Novel implementation of sampled stats is coming soon

#### Query optimizations based on statistics
