---
layout: article
title: Paper苦海(长期更新...)
comments: true
---



> 本文主要是将我日常中读到的论文进行一个大致的概括。

# Paper苦海

## Access Path Selection in a Relational Database Management System 
首先给出[论文链接](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.71.3735&rep=rep1&type=pdf)这篇文章开创新地使用了代价模型，从IO和CPU的角度出发，估算出各种方案(这里主要是unorder和interesting order两类方案)的代价，从而选择代价最小的作为最后的执行计划。

对于SQL来说，他是CPU密集型和IO密集型的集合运算，所以论文中首先给了一个用来评价一个SQL的简要代价公式：
```
COST = PAGE FETCHES + W * (RSI CALLS)
```

对于`PAGE FETCHES`，除了和各个表的cardinality相关外，还和一个因素有关————selectivity factor，其实这个selectivity factor可以认为是一个谓词组合的数学概率，这里需要说明的是这个概率是十分十分粗糙，这也是后面有些论文使用了histogram来统计相关数据的概率，具体各种情况的selectivity factor的计算可以见论文的`TABLE 1`；

有了SQL的简要代价公式和selectivity factor，我们就可以估算单表的访问代价了，如`TABLE 2`所示；

对于一个SQL来讲，他的计划空间其实是NP的，例如我们有10个表做join，你可以把t1当成第一个表，你也可以把t2、t3、t4、t5...当成第一个表，因此其实一共有`10!`中方案，那么NP-hard问题肯定不能去做穷举，因此我们选择感兴趣的组合方式，我们选择包含interesting order的index方案和unorder方案，再根据merge join和hash join对应成本的动态规划算法(相同的relations、orders，我们就认为是同一类数据展示方式，那么在动画规划中我们之前针对这种情况我们选取成本最小的，这种又将查找区间进一步缩小)选举出成本最小的方案作为最后的物理计划。

不得不感叹，计算机的先贤们真是太厉害了，即使过了40多年，该方案仍然是各大数据库中稳定使用的优化器方案之一。
