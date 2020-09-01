# HP-Lesson 3-markjenny

## 拓扑结构

使用`tiup`运行playground在个人电脑上。该拓扑包含一个`TiKV`，一个`TiDB`和一个`PD`。

## workload构建

使用`sysbench`创建数据并插入，相关脚本如下：

```
sysbench oltp_update_non_index --config-file=config --threads=32 --tables=32 --table-size=10000 prepare
```

## profile

使用如下curl方法获取集群性能数据：

```
curl http://{TiDBIP}:10080/debug/zip?seconds=60 --output debug.zip
```

并对heap profile进行分析，相关svg文件[在此](https://github.com/markjenny/markjenny.github.com/blob/master/images/heap_profile.svg)；可以看到因为我们在准备数据时插入了很多数据，导致`buildValuesListOfInsert`消耗了很多内存，但是我们还是从此入手查看是否有可以再优化的空间；

另外，`yyParse`相关的逻辑其实耗费的`cpu`和`mem`都比较多，但是由于`yyParse`是由`goyacc`自动生成的，所以这里目前没有太好的改动空间，故还是将可以优化的点转移到`TiDB`自身上。

## Issue

针对上述提出的优化思路，暂时提出了两点关于内存可以优化的地方，相关Issue的链接如下：

* [Try to reduce the reallocation of insertPlan in the buildValuesListOfInsert](https://github.com/pingcap/tidb/issues/19629)；
* [Try to optimize memory cost of the value copy in the initlialize.go](https://github.com/pingcap/tidb/issues/19627)

期待您的反馈！

