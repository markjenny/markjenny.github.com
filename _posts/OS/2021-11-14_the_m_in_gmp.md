---
layout: article
title: GMP模型中M的部分细节
comments: true
---


> 本文将对GMP中的部分M细节进行阐述，环境一起讨论指正。

# GMP模型中M的部分细节

我们以go 1.14为参考，其他应该大同小类；

首先是go程序的入口，`rt0_go`全局的m0和g0是在proc.go中的定义的，每个m都有一个g0；但是全局只有一个m0，就是程序主线程。

rt0_go函数中的args函数做了简单的参数处理；osinit也只是获取cpu核心数以及内存页大小；还比较重要的schedinit函数；并且rt0_go会进行全局m0和g0的初始化工作相关的代码如下：

```
初始化m0和g0是在rt_g0中：
// save m->g0 = g0
MOVD g, m_g0(R0)
// save m0 to g0->m
MOVD R0, g_m(g)
```

 上面可以看到m和g是互相绑定的，而m能运行的首要条件就是m必须要拥有p，所以我们在proc.go的procresize函数中可以看到下面的这种语句：

```
if _g_.m.p.ptr() == p {
 continue
}
```

这就说明m中是包含了p数据（当然这只是常规情况，如果产生了syscall并且阻塞时间较长，那么m就会从p中剥离出去），其中m结构中的p也可以看出这个p的作用：

```
p             puintptr // attached p for executing go code (nil if not executing go code)
```

proc.go中的procresize函数是重新制定P的数量，并返回一个可执行的P；这个函数一般是在程序刚开始启动，或者使用了GOMAXPROCS重新设定了P的数量后执行，这个函数会重新调整P的数量，并且返回一个可执行的P，在该函数中，还可以看到拿到了P之后，还会给P再指定一个可运行的M，相关函数如下：

```
p.m.set(mget())
p.link.set(runnablePs)
runnablePs = p
```

其中mget()函数是获取一个空闲M，这个函数比较容易理解，就不展开了，但是M的结构给我的感触更大，我之前只是知道M可以认为是一个可以运行的线程，但是内部结构总感觉知道的不是很清晰，对于我探究其他go的设计时会产生阻碍，现在知道了，他的内部结构如下：

```
type m struct {
 g0      *g     // goroutine with scheduling stack
 
 // ...暂时不管

 dlogPerM
 mOS
}
```

其中mOS变量就是各个不同系统的需要保存的变量，我们主要针对linux进行了解，linux不包含这个，更多的是我们习惯的clone等相关函数，在os_linux.go中。

如果想仔细了解linux的clone函数，可以看一下[这篇文章](https://www.cnblogs.com/charlieroro/p/14280738.htm)；

os_linux.go中创建了m，其实就是调用了clone来创建一个新的线程，但是我们在os_linux.go中看到的clone是go自己封装的一个clone，相关声明为：

```
func newosproc(mp *m) {
    // ....
}
```

具体的实现是通过汇编实现的，相关的代码格局各个平台架构不同对应不同的文件。我们以sys_linux_amd64.s中的代码为例;

首先展示sys_linux_amd64.s中的clone函数：

```
// int32 clone(int32 flags, void *stk, M *mp, G *gp, void (*fn)(void));
TEXT runtime·clone(SB),NOSPLIT,$0
 MOVL flags+0(FP), DI
 MOVQ stk+8(FP), SI
 MOVQ $0, DX
 MOVQ $0, R10

 // Copy mp, gp, fn off parent stack for use by child.
 // Careful: Linux system call clobbers CX and R11.
 MOVQ mp+16(FP), R8
 MOVQ gp+24(FP), R9
 MOVQ fn+32(FP), R12

 MOVL $SYS_clone, AX
 SYSCALL

 // In parent, return.
 CMPQ AX, $0
 JEQ 3(PC)
 MOVL AX, ret+40(FP)
 RET

 // In child, on new stack.
 MOVQ SI, SP

 // If g or m are nil, skip Go-related setup.
 CMPQ R8, $0    // m
 JEQ nog
 CMPQ R9, $0    // g
 JEQ nog

 // Initialize m->procid to Linux tid
 MOVL $SYS_gettid, AX
 SYSCALL
 MOVQ AX, m_procid(R8)

 // Set FS to point at m->tls.
 LEAQ m_tls(R8), DI
 CALL runtime·settls(SB)

 // In child, set up new stack
 get_tls(CX)
 MOVQ R8, g_m(R9)
 MOVQ R9, g(CX)
 CALL runtime·stackcheck(SB)

nog:
 // Call fn
 CALL R12

 // It shouldn't return. If it does, exit that thread.
 MOVL $111, DI
 MOVL $SYS_exit, AX
 SYSCALL
 JMP -3(PC) // keep exiting

```

可以看到golang的clone函数比linux提供的系统调用clone函数做了更多的事情，不仅使用SYSCALL指令调用了clone系统调用，如果传入clone中有m和g，还会做相关的初始化，并将新的thread的tid保存在m的procid中；
我们在该函数中可以看到如下注释：

```
 // Call fn
 CALL R12

 // It shouldn't return. If it does, exit that thread.
 MOVL $111, DI
 MOVL $SYS_exit, AX
 SYSCALL
 JMP -3(PC) // keep exiting
```

一旦调用了fn，那就不应该退出了，如果要退出那就需要调用sys_exit将111作为退出错误码，但是我没有找到该111是一个通用退出码的一个资料。

我想你肯定也看到了，上面代码的注释`It shouldn't return. If it does, exit that thread`，也就是说，新创建的M正常来讲应该不会出现传入的fn不会退出的情况，这个函数其实就是mstart函数，位置在proc.go中的mstart函数，其中细节我们暂时不去细说，只描述其中不会退出循环的原因；
mstart开始，大致函数流程如下：
mstart--->mstart1--->[schedule--->execute--->gogo--->goexit]
gogo到goexit这里有一点隐蔽，是在新建一个go的时候将goexit函数地址存储起来

```
newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
```

具体这里可以跳转到goexit执行可以说是十分hack，就是在我们创建一个新g时就偷偷地做好了；
我们知道在创建新g时是使用了newproc函数，该函数会创建一个堆栈，将参数放置在堆栈上，**另外**，会将堆栈的SP减去一个地址长度形成一个空白区域，**然后**，将之前我们在newg.pc的值放在这个空白区域；
newg.pc的值我们之前已经说过，是goexit的入口指令地址减1，一旦执行完g的函数逻辑后，就会根据函数调用执行return address，而return address是goexit的第二条指令，就是函数内部指令，这样我们就欺骗了CPU，让他以为我们之前就是在函数goexit中，从而避免了函数调用引起的堆栈变化，也就可以理解成，我们把goexit的代码直接**贴在**了每个g中的fn的末尾，此时g创建的所有局部变量goexit都可以直接调用，而不需要像函数调用那样再赋值传输了。

最后goexit中调用goexit1，goexit1中调用mcall(goexit0)实现退出当前g，并切换到g0并调用goexit0，然后g0在调用schedule()函数，实现调度循环。
整体的一个流程就简单地描述清楚了。

```
runtime.schedule---->runtime.execute
     ↑                         |
     |                         ↓
runtime.exit<-------------runtime.gogo
```

