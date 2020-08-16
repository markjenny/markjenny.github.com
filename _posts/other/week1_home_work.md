# Week 1 HomeWork

Question:

> 本地下载 `TiDB`，`TiKV`，`PD` 源代码，改写源码并编译部署以下环境：
> * 1 `TiDB`
> * 1 `PD`
> * 3 `TiKV `
> 改写后：使得 `TiDB `启动事务时，能打印出一个 “hello transaction” 的 日志

## 一、下载&&编译

下载相关依赖并依据`Makefile`编译。

## 二、运行

按照题目要求，需要起一个`PD`，三个`TiKV`和一个`TiDB`，相关的启动脚本如下：

```
# PD启动方式：
nohup bin/pd-server --name=pd1 --client-urls=http://127.0.0.1:2379 --peer-urls=http://127.0.0.1:2380 --data-dir=/home/Mark/data/pd > pd.log 2>&1 &
# TiKV启动方式：
nohup ./tikv-server --pd=127.0.0.1:2379 --data-dir=/home/Mark/data/tikv1 --log-file=/home/Mark/log/tikv1/tikv.log --addr=127.0.0.1:20160 --advertise-addr=127.0.0.1:20160 > tikv1.log 2>&1 &
nohup ./tikv-server --pd=127.0.0.1:2379 --data-dir=/home/Mark/data/tikv2 --log-file=/home/Mark/log/tikv2/tikv.log --addr=127.0.0.1:20161 --advertise-addr=127.0.0.1:20161 > tikv2.log 2>&1 &
nohup ./tikv-server --pd=127.0.0.1:2379 --data-dir=/home/Mark/data/tikv3 --log-file=/home/Mark/log/tikv3/tikv.log --addr=127.0.0.1:20162 --advertise-addr=127.0.0.1:20162 > tikv3.log 2>&1 &
# TiDB启动方式：
nohup bin/tidb-server --store=tikv --path=127.0.0.1:2379 > tidb.log 2>&1 &
```

运行正常，可以实现SQL相关操作，本地现有数据：

![tidb_work_on](https://github.com/markjenny/markjenny.github.com/blob/master/images/tidb_work_on%20.png)

## 三、打印出“hello transaction” 

相关详细问题为：

> 使得 TiDB 启动事务时，能打印出一个 “hello transaction” 的 日志

通过读取相关的官方博客文章[SQL的一生](https://pingcap.com/blog-cn/tidb-source-code-reading-3/)，可以找到相关的入口函数，但是由于代码还是有些复杂，快速找到启动事务的地方略显负责；因此变使用`TiDB`边查看相关日志，发现`2pc.go`文件中在数据提交时查看到`checkSchemaValid`的相关日志，因此确定该文件和事务提交是相关的，查看相关文档[乐观锁事务](https://pingcap.com/blog-cn/best-practice-optimistic-transaction/)将事务相关首先规划在`TiDB`文件夹中；

对于入口文件`session.go`中查看和kv相关的代码及逻辑，找到函数`InitTxnWithStartTS`，该函数是通过全局分配的StartTS来初始化事务，该函数部分代码如下：

```
// InitTxnWithStartTS create a transaction with startTS.
func (s *session) InitTxnWithStartTS(startTS uint64) error {
	if s.txn.Valid() {
		return nil
	}

	// no need to get txn from txnFutureCh since txn should init with startTs
	txn, err := s.store.BeginWithStartTS(startTS)
	if err != nil {
		return err
	}
	txn.SetVars(s.sessionVars.KVVars)
	s.txn.changeInvalidToValid(txn)
	err = s.loadCommonGlobalVariablesIfNeeded()
	if err != nil {
		return err
	}
	return nil
}
```

其中的`txn, err := s.store.BeginWithStartTS(startTS)`即是通过`startTS`来创建transaction；相关代码改动如下：

![hello transaction](https://github.com/markjenny/markjenny.github.com/blob/master/images/hello_transaction.png)

相关测试代码输出：

![hello transaction output](https://github.com/markjenny/markjenny.github.com/blob/master/images/hello_transaction_output.png)

从上图的输出可以看出，每次有一个begin或begin后紧接的查询SQL后，都会看起一个transaction（`Hello Execute`的输出为`sesssion.go`中的Execute执行入口）。

之所在`Execute`入口处添加该行输出，是因为我之前已经添加完`hello transaction`的相关代码后发现一直打印改行输出但是我却没有进行相关的SQL操作，比较迷惑（*这里脑袋一时糊涂没有想到后台会进行数据统计相关的SQL查询*），打印后发现后台会定时执行统计SQL用于后期计算。



