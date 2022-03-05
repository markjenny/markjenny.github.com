---
layout: article
title: 关于Go的临时笔记（长期更新）
comments: true
---


> 本文主要是记录自己平时关注的一些细小知识，但是单独出一篇blog又特别水的这类知识，长期更新。

-----

# 临时使用

对于**放一半**的流程是这样的：如果本地的runnext和local runq都满了的话，就需要将一部分g放到global runq中，选取的原则是将local runq中后半部分和runnext中的g组合一起放到global runq中的尾部，之所以选择后半部分，我的理解是：对于当前的P来说后半部分和runnext中被赶下来的g，即使在本地P中也是要**往后稍稍**的角色，所以索性就把他们放到global runq中，前半部分的已经等了很久了，P必须要照顾一下他们的情绪。

对于**取一半**的流程，如果是从全局队列中获取一半，那么是从global runq中拿出`(全局队列g总数/gomaxprocs)+1`个，但是不能超过local runq的一半——128个，这里我们可以这样理解： 对于全局队列来说我们肯定要先获取其位于队首那部分的g，因为他们已经排队较久了，如果拿到其他部分的话那么对于P来说他肯定要先执行自己local runq上的g，这样global runq上的g又可能会有等待的情况，这样对于global runq队首元素来说是不公平的，所以我们在从global runq中取g到自己的local runq中时，要取global runq的前面的元素。如果这样你能想通，那么你肯定也理解为什么从取出来的这些g队列中取第一个来进行执行。

 当然你肯定也听过steal，就是偷去其他p中的g队列放到自己这里来执行，如果你问：“为什么对于其他p的runq也是取偷取前面的g并且执行所偷的g队列的最后一个元素呢？”，抱歉我也百思不得其解，可能是为了参照拿取global runq的逻辑，但是为什么又拿了最后一个g去执行。。。这里确实有点迷。

对于**取一半**的流程，似乎不同的场景下获取的逻辑不一样：

对于`runqsteal`来说，他每次是偷其他的P的前半部分g放到自己的P的队列中；但是是将最后一个g拿出啦进行执行，这里和其他情况下通常取队列头部的g进行执行的逻辑不通，但是为什么这样不是很清楚，有点迷；

但是对于`globrunq`，额。。。刚才发现也是把前半部分拿过来；并把前半部分中的第一个g关联到m上进行执行；

这其实也是正确的，因为前半部分的g是等待时间比较长的了，并且后半部分的g可以更加地利用局部性原理。

---------------

看了一下systemstack的逻辑流程，可以发现其中是将对战切换成了g0；函数位置在`asm_amd64.s:333`的`runtime.systemstack`；

`goready`会切换到g0然后运行`ready`程序，将要恢复的g放到当前关联p的对应接列中；他的逻辑像新生成一个g的时候是一样的，想放到`runnext`指针中，然后放到localrunq，最后放到global runq中。

----

gopark函数会将当前g的相关状态保存后，m直接放弃当前的g，直接关联g0并执行`park_m`函数来schedule获取可执行g；

相关代码注释如下：

```
 func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	if reason != waitReasonSleep {
		checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
	}
	mp := acquirem()
	gp := mp.curg
	status := readgstatus(gp)
	if status != _Grunning && status != _Gscanrunning {
		throw("gopark: bad g status")
	}
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp)
	// can't do anything that might move the G between Ms here.
	// 上面的代码是对当前的m和g进行一些状态记录，下面的mcall涉及到具体切换
	mcall(park_m)
}

// 下面代码是asm_amd64.s中的mcall函数
// void mcall(fn func(*g))
// Switch to m->g0's stack, call fn(g).
// Fn must never return. It should gogo(&g->sched)
// to keep running g.
TEXT runtime·mcall(SB), NOSPLIT|NOFRAME, $0-8
	// Save caller state in g->sched
	// 将当前寄存器信息保存到g->sched的gobuf中，我们知道gobuf是当前g的现场，下面的保存数据都是我们熟悉的sp、bp、pc等
	MOVD	RSP, R0
	MOVD	R0, (g_sched+gobuf_sp)(g)
	MOVD	R29, (g_sched+gobuf_bp)(g)
	MOVD	LR, (g_sched+gobuf_pc)(g)
	MOVD	$0, (g_sched+gobuf_lr)(g)
	MOVD	g, (g_sched+gobuf_g)(g)

	// Switch to m->g0 & its stack, call fn. 
	MOVD	g, R3
	MOVD	g_m(g), R8
	MOVD	m_g0(R8), g
	// BL即跳转到runtime·save_g来保存g的信息
	BL	runtime·save_g(SB)
	CMP	g, R3
	BNE	2(PC)
	B	runtime·badmcall(SB)
	MOVD	fn+0(FP), R26			// context
	MOVD	0(R26), R4			// code pointer
	MOVD	(g_sched+gobuf_sp)(g), R0
	MOVD	R0, RSP	// sp = m->g0->sched.sp
	MOVD	(g_sched+gobuf_bp)(g), R29
	MOVD	R3, -8(RSP)
	MOVD	$0, -16(RSP)
	SUB	$16, RSP
	BL	(R4)
	B	runtime·badmcall2(SB)

```

上面代码中的`BL	runtime·save_g(SB)`其实我在刚看mcall代码中就有点不是很明白，因为我们在mcall的上半部分已经保存了g的相关信息，并在gopark中也保存了部分信息，为什么还需要进行保存呢，于是我查看了save_g中的代码看了一下，发现这里应该是和cgo相关的（目前cgo不是我关注的点，所以不去了解），下面展示一下代码：

```
TEXT runtime·save_g(SB),NOSPLIT,$0
	MOVB	runtime·iscgo(SB), R0
	CMP	$0, R0
	// 如果iscgo的返回值R0等于$0，那么就跳转到nocgo处执行
	BEQ	nocgo

	MRS_TPIDR_R0
#ifdef GOOS_darwin
	// Darwin sometimes returns unaligned pointers
	AND	$0xfffffffffffffff8, R0
#endif
	MOVD	runtime·tls_g(SB), R27
	ADD	R27, R0
	MOVD	g, 0(R0)

nocgo:
	RET
```

所以如果你是arm64架构并且不设计到cgo，那么这里应该没有什么作用，并且通过调试我们也可以发现直接RET了，如下图所示：

![save_g直接返回](https://github.com/markjenny/markjenny.github.com/blob/master/images/save_g_ret.png)

mcall后面暂时不是我感兴趣的了，就是切换到g0后直接调用mcall中的park_m进行执行，就是选取可以执行的g。

-----------------

sysmon中的retake函数会执行handoff来解绑m和p；然后再调用startm来让p重新绑定新的m

---------

pageAlloc分配大页的函数中，其中有维持这些大页的结构；pāgeAlloc结构经过多个版本的变化，从: freelist->treap->radix tree，查询时间复杂度越来越低，结构越来越复杂。
