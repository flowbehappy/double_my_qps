# Double My QPS

成员: @flowbehappy, @iosmanthus, @pingyu, @tangenta (in alphabetical order)

## 项目介绍

一键让 TiDB 的 QPS 翻倍，`set tidb_double_my_qps = 'ON';`

## 背景&动机

众所周知，TiDB 具有良好的扩展性，可以通过简单扩容集群来获得更高的 QPS。但是在某些场景，仍然无法完美的满足需求：

1. 双 11 马上到了，你去找老板要求给 TiDB 集群扩容一倍，以应对预期翻倍的流量。老板的回复：采购来不及了，想办法撑一下吧。
2. 你的应用总是有各种突发十几分钟的热点，即使上了 TiDB Cloud，等 TiKV 扩容出来热点可能已经消退了。
3. 定时跑批任务，但又是 TP 负载（比如银行结算），用户只关心啥时候跑完，不关心每一条 SQL 的耗时，能否不扩容的情况下跑快点？
4. 穷，想用更少的机器跑更多的负载，SQL 耗时高点可以忍，QPS 能打高可以撑住业务就行。

**核心思路**：在不增加 TiDB 节点的前提下，通过各种优化，以及适当牺牲其他指标，换取尽可能高的 QPS。

## 项目设计

本项目包含两种优化方向：

1. 通用的，特别是可以提高 QPS 的
2. 会劣化其他性能指标的（比如 P99 latency），但是可以提高 QPS 的

为了方便对比效果，我们把这些优化打包到一个开关中：`set tidb_double_my_qps = 'ON';`

### Pipeline 一切
让数据库读写操作的各个过程，尽可能并行执行，充分利用 CPU。已知可以优化的点：

1. https://github.com/tikv/tikv/issues/8752 Fully pipelined push-based query engine of TiKV

### 提高 CPU 的使用效率

以 TiDB Cloud 的小集群 1 TiDB（8C） + 3 TiKV（8C，gp3 EBS） 为例，我们发现当并发数上去之后，系统的瓶颈是 CPU，比如 sysbench oltp_point_select，TiDB 100%，TiKV 30%。所以优化的重点之一在于让 TiDB 和 TiKV 能用更少的 CPU cycle 干更多的活。在 AP 的世界中，主要有两种流派来达到类似的效果：

1. Code Generation
2. Vectorization

方法 1 效果预期不明显，也太复杂，我们主要看方法 2。核心点之一就是攒批处理，即每个算子每次处理多条记录，而非一条记录。实际上 TiDB、TiKV 已经做了向量化处理，但是由于 TP 的每条 SQL 数据量通常较小，攒批处理的效果没有 AP 这么明显。换一个角度，既然单条 SQL 内部的数据小，那是不是可以把相同执行计划的多条 SQL 一起攒批处理呢？沿着这个思路，已知有这些问题需要处理：

1. 不同 session 参数不同，执行逻辑也不一样
2. 不同的 SQL 的时间戳 ts 不一样
3. 查询的过滤条件不一样
4. 不同 SQL 的数据不能被聚合、join

### 去掉非核心逻辑
在业务关键时刻，用户需要所有资源都投入在最核心的业务上，一些平时需要但不致命的逻辑可以不跑。比如收集统计数据的逻辑可以关掉。

### 面向 QPS 的调参

其他都可以不要，只要正确性、高可用和 QPS


## 测试负载

sysbench


