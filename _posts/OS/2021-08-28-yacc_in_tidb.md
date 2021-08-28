---
layout: article
title: tidb中的语法解析
comments: true
---


> 本文主要是在官方文档及网络文档辅助下，写下自己学习SQL的语法解析逻辑的一点点学习经验。

------

# tidb中的SQL语法分析学习

之前看了很久的lex && yacc，但是由于本人对编译原理不是很懂（留下了没有技术的泪水。。。），所以看了几次都觉得了解的不是很深入；这次再学习tidb的SQL语法分析时感觉有一些小领悟。

之前在学习的时候，有一些文章如：[Lex和Yacc——Lex学习](https://niyanchun.com/yacc-learning.html)以及[TiDB源码阅读笔记（二） 简单理解 Lex & Yacc](https://zhuanlan.zhihu.com/p/165488348)写的非常好，我们知道yacc会按照你写的语法规则来对token列表进行分析，生成最后的AST；

在tidb中，根据parser.y的内容，最后都会生成一个SelectStmt、DropTableStmt、DeleteStmt等等的ast.Node，ast.Node就可以理解成是根据yacc规则而生成的非终结符；我们知道yacc是根据BNF来一步一步将满足条件的终结符归约成最后的一个非终结符，这个非终结符就可以理解成是tidb的parser.go根据SQL最后生成的SelectStmt、DropTableStmt、DeleteStmt这种非终结符节点。

yacc解析的规则就可以理解成是一个堆栈操作，如果你看过或者写过*如何用栈完成一个简易计算器功能*的相关题目，你就可以理解yacc的规约过程，如果堆栈中栈顶的几个元素可以规约成一个非终结符，那么他们就会弹出来后，按照之前在yacc中设置的语法规则生成特定的非终结符后，再压入堆栈并为下次规约做准备。

我们以**JoinTable**中的语法为例进行说明：

```
JoinTable:
	/* Use %prec to evaluate production TableRef before cross join */
   // 这个连接是对prec的解释，https://stackoverflow.com/questions/5330541/what-does-prec-mean-here-in-yacc
   // 可以认为【TableRef CrossOpt TableRef】是一个左结合的操作，这样对于select a join b join c 就可以理解成是select (a join b) join c了
	TableRef CrossOpt TableRef %prec tableRefPriority
	{
    	$$ = &ast.Join{Left: $1.(ast.ResultSetNode), Right: $3.(ast.ResultSetNode), Tp: ast.CrossJoin}
    }
|	TableRef CrossOpt TableRef "ON" Expression
	// 上一行是一个规约规则，即堆栈从顶端开始的第一元素满足【TableRef CrossOpt TableRef "ON" Expression】这几个token排列规则，那么就可以规约成JoinTable并再次放入到堆栈了去继续规约
  // 例如【select * from t1 join t2 on t1.id = t2.di】那么其中的【t1 join t2 on t1.id = t2.id】就会规约成JoinTable，就会最后成为SelectStmt节点的From数据
  {
    
		on := &ast.OnCondition{Expr: $5}
		$$ = &ast.Join{Left: $1.(ast.ResultSetNode), Right: $3.(ast.ResultSetNode), Tp: ast.CrossJoin, On: on}
	}
|	TableRef JoinType OuterOpt "JOIN" TableRef "ON" Expression
	{
    // 下面两行是满足规约条件后的逻辑处理，代码最上面的两行是使用goyacc生成的parser.go中的代码，可以看到最后成了一个ast.Join结构并放入了栈顶元素中
    // 其中$1就是对应规则中的第一个符号，例如$7就是对应Expression符号，这是说明按照何种条件进行关联join，因此$7这个变量就是组成ast.OnCondition结构的核心数据
    // 剩下的ast.Join中的Left和Right数据成员就是对应$1————第一个TableRef、$5————第二个TableRef；按照规则生成的JoinTable就是下面的【$$】变量，放回堆栈进行后续的其他语法解析操作
    // on := &ast.OnCondition{Expr: yyS[yypt-0].expr}                                                                      
    // parser.yyVAL.item = &ast.Join{Left: yyS[yypt-4].item.(ast.ResultSetNode), Right: yyS[yypt-2].item.(ast.ResultSetNode), Tp: ast.CrossJoin, On: on}
		on := &ast.OnCondition{Expr: $7}
		$$ = &ast.Join{Left: $1.(ast.ResultSetNode), Right: $5.(ast.ResultSetNode), Tp: $2.(ast.JoinType), On: on}
	}
```

我们在TestDMLStmt测试函数中，测试了【SELECT * from t1, t2, t3】，其实这也是一种join操作，但是没有走入JoinTable中，而是直接解析成了join操作，相关的调试结果如下：
```
TableRefs:
	EscapedTableRef
	{
		if j, ok := $1.(*ast.Join); ok {
			// if $1 is Join, use it directly
			$$ = j
		} else {
			$$ = &ast.Join{Left: $1.(ast.ResultSetNode), Right: nil}
		}
	}
|	TableRefs ',' EscapedTableRef
	{
		/* from a, b is default cross join */
        // 典型的规约做法，将栈顶元素[$1, $2, $3分别应用在下面的规约公式中，得到的结果$$再次放入到BNF堆栈中]
        // 针对join的连表操作，我们可以通过调试TestDMLStmt案例来查看【SELECT * from t1, t2, t3】
        // 我们可以看到这就是一个递归成SelectStmt结构的ast
        // (dlv) p parser.result
        // []github.com/pingcap/tidb/parser/ast.StmtNode len: 1, cap: 1, [
        // 	*github.com/pingcap/tidb/parser/ast.SelectStmt {
        // 		dmlNode: (*"github.com/pingcap/tidb/parser/ast.dmlNode")(0xc0002d7380),
        // 		SelectStmtOpts: *(*"github.com/pingcap/tidb/parser/ast.SelectStmtOpts")(0xc0003147b0),
        // 		Distinct: false,
        // 		From: *(*"github.com/pingcap/tidb/parser/ast.TableRefsClause")(0xc000292040),
        // 		Where: github.com/pingcap/tidb/parser/ast.ExprNode nil,
        // 		Fields: *(*"github.com/pingcap/tidb/parser/ast.FieldList")(0xc0003147e0),
        // 		GroupBy: *github.com/pingcap/tidb/parser/ast.GroupByClause nil,
        // 		Having: *github.com/pingcap/tidb/parser/ast.HavingClause nil,
        // 		OrderBy: *github.com/pingcap/tidb/parser/ast.OrderByClause nil,
        // 		Limit: *github.com/pingcap/tidb/parser/ast.Limit nil,
        // 		TableHints: []*github.com/pingcap/tidb/parser/ast.TableOptimizerHint len: 0, cap: 0, nil,
        // 		IsInBraces: false,},
        // ]
        // (dlv) p parser.result[0].From
        // *github.com/pingcap/tidb/parser/ast.TableRefsClause {
        // 	node: github.com/pingcap/tidb/parser/ast.node {text: ""},
        // 	TableRefs: *github.com/pingcap/tidb/parser/ast.Join {
        // 		node: (*"github.com/pingcap/tidb/parser/ast.node")(0xc0003201c0),
        // 		Left: github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.Join) ...,
        // 		Right: github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.TableSource) ...,
        // 		Tp: CrossJoin (1),
        // 		On: *github.com/pingcap/tidb/parser/ast.OnCondition nil,},}
        // (dlv) p parser.result[0].From.TableRefs.Left
        // github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.Join) *{
        // 	node: github.com/pingcap/tidb/parser/ast.node {text: ""},
        // 	Left: github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.Join) *{
        // 		node: (*"github.com/pingcap/tidb/parser/ast.node")(0xc0003200c0),
        // 		Left: github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.TableSource) ...,
        // 		Right: github.com/pingcap/tidb/parser/ast.ResultSetNode nil,
        // 		Tp: 0,
        // 		On: *github.com/pingcap/tidb/parser/ast.OnCondition nil,},
        // 	Right: github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.TableSource) *{
        // 		node: (*"github.com/pingcap/tidb/parser/ast.node")(0xc000320100),
        // 		Source: github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.TableName) ...,
        // 		AsName: (*"github.com/pingcap/tidb/parser/model.CIStr")(0xc000320120),},
        // 	Tp: CrossJoin (1),
        // 	On: *github.com/pingcap/tidb/parser/ast.OnCondition nil,}
        // (dlv) p parser.result[0].From.TableRefs.Left.Left
        // github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.Join) *{
        // 	node: github.com/pingcap/tidb/parser/ast.node {text: ""},
        // 	Left: github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.TableSource) *{
        // 		node: (*"github.com/pingcap/tidb/parser/ast.node")(0xc000320080),
        // 		Source: github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.TableName) ...,
        // 		AsName: (*"github.com/pingcap/tidb/parser/model.CIStr")(0xc0003200a0),},
        // 	Right: github.com/pingcap/tidb/parser/ast.ResultSetNode nil,
        // 	Tp: 0,
        // 	On: *github.com/pingcap/tidb/parser/ast.OnCondition nil,}
        // (dlv) p parser.result[0].From.TableRefs.Left.Left.Left.Source
        // github.com/pingcap/tidb/parser/ast.ResultSetNode(*github.com/pingcap/tidb/parser/ast.TableName) *{
        // 	node: github.com/pingcap/tidb/parser/ast.node {text: ""},
        // 	Schema: github.com/pingcap/tidb/parser/model.CIStr {O: "", L: ""},
        // 	Name: github.com/pingcap/tidb/parser/model.CIStr {O: "t1", L: "t1"},
        // 	DBInfo: *github.com/pingcap/tidb/parser/model.DBInfo nil,
        // 	TableInfo: *github.com/pingcap/tidb/parser/model.TableInfo nil,
        // 	IndexHints: []*github.com/pingcap/tidb/parser/ast.IndexHint len: 0, cap: 0, nil,
        // 	PartitionNames: []github.com/pingcap/tidb/parser/model.CIStr len: 0, cap: 0, nil,}
		$$ = &ast.Join{Left: $1.(ast.ResultSetNode), Right: $3.(ast.ResultSetNode), Tp: ast.CrossJoin}
	}

```
