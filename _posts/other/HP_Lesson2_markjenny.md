# HP_Lesson2

## 基础信息

1. 部署环境机器配置

   所部署的机器均为腾讯云主机，相关云主机的配置如下：

   > 1核(AMD EPYC 7K62 48-Core Processor)，2GB内存，50G SATA磁盘存储部署数据。

2. 拓扑结构

   如图所示：

   ![图一](https://github.com/markjenny/markjenny.github.com/blob/master/images/部署拓扑结构.png)

3. 调整过后的TiDB和TiKV配置

   由于条件所限，只申请了4台机器；而TiKV和TiDB不建议部署在同一台机器上；因为TiDB和TiKV都设计到CPU密集型计算。并且TiKV对内存要求较高，所以在初期部署时就直接采用了3PD、3TiKV和1TiDB的拓扑，并将TiKV和TiDB分开。

## sysbench测试

1. 测试脚本及相关数据

   * 导入数据

     ```
     # 导入32个表，每个表100000条数据
     sysbench oltp_update_non_index --config-file=config --threads=32 --tables=32 --table-size=10000 prepare
     ```

   * 查询数据及结果

     ```
     # 查询数据
     sysbench oltp_point_select --config-file=config --threads=128 --tables=32 --table-size=10000 run   
     # 结果展示：
     thds: 128 tps: 13029.46 qps: 13029.46 (r/w/o: 13029.46/0.00/0.00) lat (ms,95%): 23.95 err/s: 0.00 reconn/s: 0.00
     ```

   * 更新数据及结果

     ```
     # 更新数据
     sysbench --config-file=config oltp_update_index --threads=128 --tables=32 --table-size=10000 run
     # 结果展示：
     128 tps: 2311.03 qps: 2311.03 (r/w/o: 0.00/2311.03/0.00) lat (ms,95%): 89.16 err/s: 0.00 reconn/s: 0.00
     ```

2. 关键指标的监控截图

   * 数据查询相关指标

     数据查询时TiKV的CPU和QPS监控参数如下：

     ![图二](https://github.com/markjenny/markjenny.github.com/blob/master/images/sysbench_查询数据_CPU_IO.jpg)

     ![图三](https://github.com/markjenny/markjenny.github.com/blob/master/images/sysbench_查询数据_QPS.jpg)

     从上图可以看出查询数据的QPS峰值可以到1.3K；IO开销较小；

     数据查询时TiKV的grpc相关监控参数如下：

     ![图四](https://github.com/markjenny/markjenny.github.com/blob/master/images/sysbench_查询数据_grpc.jpg)

     可以看到gprc的qps峰值可以到2.8K，其中大部分都是`kv_get`，直接获取相关参数；

     数据查询时TiDB的Query Summary监控参数如下：

     ![图五](https://github.com/markjenny/markjenny.github.com/blob/master/images/sysbench_查询数据_Tidb_summary.jpg)

     在上面的多张监控参数截图中，需要说明的是，19:10左右时进行了数据插入，都有由于没有更换成乐观锁模式，导致大量的阻塞、同步操作，影响了相关的性能。

   * 数据更新相关指标

     数据更新时TiKV的CPU和QPS监控参数如下：

     ![图六](https://github.com/markjenny/markjenny.github.com/blob/master/images/sysbench_更新数据_kv_details.png)

     可以看到QPS比查询数据时要低大约低20%左右的比例，在更新阶段kv_pessimistic_lock和coprocessor花费了大量了时间，因为在更新时为了保证ACID特定，悲观锁模式会进行大量的阻塞，导致QPS降低；其中coprocessor也造成了QPS下降。

     数据更新时TiKV的grpc监控参数如下：

     ![图七](https://github.com/markjenny/markjenny.github.com/blob/master/images/sysbench_更新数据_grpc.jpg)

     从上图中可以看到，grpc的qps并没有太明显的下降，但是TiKV的qps下降却比较明显，这说明瓶颈是在维护pessimistic_lock时产生的阻塞拖累了总的qps。

     数据更新时TiDB的Query Summary监控参数如下：

     ![图八](https://github.com/markjenny/markjenny.github.com/blob/master/images/sysbench_更新数据_tidb.jpg)

## go-ycsb

1. 测试脚本及相关数据

   * 导入数据

     ```
     ./bin/go-ycsb load mysql -P workloads/workloada -p recordcount=300000 -p mysql.host=172.27.0.8 -p mysql.port=4000 --threads 16
     ```

   * 运行

     ```
     ./bin/go-ycsb run mysql -P workloads/workloada -p recordcount=300000 -p mysql.host=172.27.0.8 -p mysql.port=4000 --threads 16
     ```

   * 运行结果

     ```
     # workloads/workloada 结果
     Run finished, takes 1.074185347s
     READ   - Takes(s): 1.1, Count: 524, OPS: 492.4, Avg(us): 3221, Min(us): 799, Max(us): 22979, 99th(us): 15000, 99.9th(us): 23000, 99.99th(us): 23000
     UPDATE - Takes(s): 1.0, Count: 468, OPS: 450.2, Avg(us): 28708, Min(us): 13637, Max(us): 51752, 99th(us): 49000, 99.9th(us): 52000, 99.99th(us): 52000
     
     # workloads/workloadb 结果
     READ   - Takes(s): 0.4, Count: 951, OPS: 2553.9, Avg(us): 4045, Min(us): 797, Max(us): 38038, 99th(us): 15000, 99.9th(us): 39000, 99.99th(us): 39000
     UPDATE - Takes(s): 0.3, Count: 41, OPS: 118.7, Avg(us): 24646, Min(us): 16311, Max(us): 41420, 99th(us): 42000, 99.9th(us): 42000, 99.99th(us): 42000
     
     # workloadb相比workloada涉及到更多的update，那么需要有pessimistic lock，这会略微影响它的时间；
     
     # workloads/workloadc 结果
     Run finished, takes 331.406872ms
     READ   - Takes(s): 0.3, Count: 992, OPS: 3306.5, Avg(us): 4762, Min(us): 1095, Max(us): 38680, 99th(us): 18000, 99.9th(us): 39000, 99.99th(us): 39000
     
     # workloads/workloadd 结果
     Run finished, takes 395.594312ms
     INSERT - Takes(s): 0.4, Count: 56, OPS: 156.8, Avg(us): 30418, Min(us): 18458, Max(us): 51737, 99th(us): 52000, 99.9th(us): 52000, 99.99th(us): 52000
     READ   - Takes(s): 0.4, Count: 936, OPS: 2430.1, Avg(us): 4074, Min(us): 793, Max(us): 23281, 99th(us): 18000, 99.9th(us): 24000, 99.99th(us): 24000
     
     # workloads/workloade 结果
     Run finished, takes 800.660777ms
     INSERT - Takes(s): 0.8, Count: 61, OPS: 79.7, Avg(us): 10222, Min(us): 1573, Max(us): 42484, 99th(us): 43000, 99.9th(us): 43000, 99.99th(us): 43000
     SCAN   - Takes(s): 0.8, Count: 931, OPS: 1209.6, Avg(us): 11696, Min(us): 1878, Max(us): 34718, 99th(us): 31000, 99.9th(us): 35000, 99.99th(us): 35000
     
     # workloads/workloadf 结果
     Run finished, takes 1.352906607s
     READ   - Takes(s): 1.3, Count: 992, OPS: 739.1, Avg(us): 3215, Min(us): 786, Max(us): 27939, 99th(us): 13000, 99.9th(us): 28000, 99.99th(us): 28000
     READ_MODIFY_WRITE - Takes(s): 1.3, Count: 506, OPS: 385.5, Avg(us): 35109, Min(us): 17625, Max(us): 70693, 99th(us): 58000, 99.9th(us): 71000, 99.99th(us): 71000
     UPDATE - Takes(s): 1.3, Count: 506, OPS: 385.5, Avg(us): 31904, Min(us): 15677, Max(us): 69436, 99th(us): 54000, 99.9th(us): 70000, 99.99th(us): 70000
     ```

2. 关键指标的监控截图（取最后workloadf的监控结果）

   * 各个workload下的相关指标

     TiKV的CPU及QPS监控参数如下：

     ![图九](https://github.com/markjenny/markjenny.github.com/blob/master/images/yscb_workload_tikv_details.jpg)

     TiKV的grpc相关监控参数如下：

     ![图十](https://github.com/markjenny/markjenny.github.com/blob/master/images/yscb_workload_rpc.jpg)

     TiDB的Query Summary监控参数如下：

     ![图十一](https://github.com/markjenny/markjenny.github.com/blob/master/images/yscb_workload_tidb.jpg)

     综合来讲，更新操作或插入操作的存在会拉低整体qps，这主要是因为在数据插入时，由于kv内存的原因，内存已趋近于满载，但是IO和CPU还没有满载的情况；这导致内存在目前拓扑中成为了该系统的瓶颈，并且各个更新的workload中如果涉及到较多的update，那么需要有pessimistic lock，这会略微影响它的时间；

     更新或插入的数据越多，产生的耗时越多，其中如上图grpc中涉及到写数据的话会有prewrite，可以看到prewrite耗时较长，这样就会耗费kv的内存，又由于最后需要落盘，所以会有较多的IO；又由于内存较少（2GB），导致写操作一上来系统资源就会不足，qps变低。并且需要一直解决锁的问题（可以看grpc的图片）。

## go-tpc

1. 测试脚本及相关结果

   * 导入数据

     ```
     ./bin/go-tpc tpcc -H 172.27.0.8 -P 4000 -D tpcc --warehouses 8 prepare
     ```

   * 运行

     ```
     ./bin/go-tpc tpcc -H 172.27.0.8 -P 4000 -D tpcc --warehouses 8 run --time 5m  --threads 4 # 测试5分钟，4个线程
     ```

   * 运行结果

     ```
     tpmC: 205.3
     ```

2. 关键指标的监控截图

   TiKV的CPU及QPS监控参数如下：

   ![图十二](https://github.com/markjenny/markjenny.github.com/blob/master/images/tpcc_tikv_details_cpu.jpg)

   ![图十三](https://github.com/markjenny/markjenny.github.com/blob/master/images/tpcc_tikv_detail_qps.jpg)

   TiKV的grpc监控参数如下：
   ![图十四](https://github.com/markjenny/markjenny.github.com/blob/master/images/tpcc_tikv_grpc.jpg)

   TiDB的Query Summary监控参数如下：

   ![图十五](https://github.com/markjenny/markjenny.github.com/blob/master/images/tpcc_tidb.jpg)

   上面四张监控图片明显地展示出了update拉低qps的现象；update或insert之所以会这么慢出了之前说明的原因，还有一点可以通过[svg文件](https://github.com/markjenny/markjenny.github.com/blob/master/images/profiling_4_4_tidb_172_27_0_8_4000809682379.svg)分析出来，其中内存也是一个很重要的原因，要频繁的malloc，那么没有足够的内存肯定会进行落盘操作；

3. 说明

   tpch的相关准备脚本及结果

   ```
   ./bin/go-tpc tpch prepare -H 172.27.0.8  -P 4000 -D tpch --sf 1 --analyze # sf是数据量内存
   ./bin/go-tpc tpch run -H 172.27.0.8 -P 4000 -D tpch --sf 1
   ```

   但是由于服务器内存（2GB）的限制，导致即使导入1G的数据也会产生OOM，而SF又不能支持小数，导致实验没有完成。

   ```
   invalid argument "0.5" for "--sf" flag: strconv.ParseInt: parsing "0.5": invalid syntax
   ```

## 结论

针对该环境、该拓扑结构下存在的性能瓶颈进行总结：

1. 内存限制，TiKV依赖大量内存来保证高性能，一旦性能成为瓶颈，会导致总体qps下降较多；
2. processor和pessimistic lock会导致阻塞严重，影响总体qps；
3. 通过[svg文件](https://github.com/markjenny/markjenny.github.com/blob/master/images/profiling_4_4_tidb_172_27_0_8_4000809682379.svg)可以发现sql解析其实也是整体流程中比较耗时的一环，TiDB使用了go-yacc的语法分析，依赖于有限状态自动机，后期是否可以通过手动处理来提高性能；

## 建议

tiup一键部署的官方文档中，建议加上grafana的相关连接，因为tiup不仅一键部署了各个server，而且还配套的创建了dashboard，在学习过程中误以为tiup没有创建相关的dashboard又学习官方文档学习如何创建dashboard，导致重点有些偏移到学习grafana中了。
