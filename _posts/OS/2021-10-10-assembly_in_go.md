---
layout: article
title: Go中的汇编语言
comments: true
---


> 本文将在一篇汇编文章的基础上，进一步进行简略汇总，方便已经有汇编语言经验(特别是x86)的相关同学可以进行整体快速掌握。
------

# Go中的汇编语言

首先需要说明本文是在曹大的博文[Go 系列文章3 ：plan9 汇编入门](https://xargin.com/plan9-assembly/)的基础上再次进行整体汇总，方便我自己和有一些汇编基础的同学可以更加整体地了解plan9和x86汇编语言的一些底层差异。

## 整体概览

如果要了解plan9，那么有4个伪寄存器你必须要了解，他们和你之前了解的会有一些出入，那么我们直接引出相关伪寄存器指向位置：
```
-----------------                                           
current func arg0                                           
----------------- <----------- FP(pseudo FP)                
caller ret addr                                            
+---------------+                                           
| caller BP(*)  |                                           
----------------- <----------- SP(pseudo SP，实际上是当前栈帧的 BP 位置)
|   Local Var0  |                                           
-----------------                                           
|   Local Var1  |                                           
-----------------                                           
|   Local Var2  |                                           
-----------------                -                          
|   ........    |                                           
-----------------                                           
|   Local VarN  |                                           
-----------------                                           
|               |                                           
|               |                                           
|  temporarily  |                                           
|  unused space |                                           
|               |                                           
|               |                                           
-----------------                                           
|  call retn    |                                           
-----------------                                           
|  call ret(n-1)|                                           
-----------------                                           
|  ..........   |                                           
-----------------                                           
|  call ret1    |                                           
-----------------                                           
|  call argn    |                                           
-----------------                                           
|   .....       |                                           
-----------------                                           
|  call arg3    |                                           
-----------------                                           
|  call arg2    |                                           
|---------------|                                           
|  call arg1    |                                           
-----------------   <------------  hardware SP 位置           
| return addr   |                                           
+---------------+                                           
```
有了这个图我们再引入各个伪寄存器的定义，就可以更好地去理解相关伪寄存器的作用：
* FP: Frame pointer: arguments and locals.
* PC: Program counter: jumps and branches.
* SB: Static base pointer: global symbols.
* SP: Stack pointer: top of stack.

## 详细说明
那么现在来详细说一下：首先PC不用多说，SB是定位全局变量和函数；
我们再来详细讨论一下伪FP和伪SP这两个寄存器：
FP是callee查找其参数和返回值的地方，参数和返回值是倒序插入的，就是为了通过FP读取时是正序的，因为堆栈就是**先入后出、后入先出**；SP是用来读取局部变量的，但是需要注意的是伪SP是指向callee的最底部，所以如果要访问局部变量的话就必须将其指向的位置减去局部变量占用的空间长度；

文章中说到了不要使用伪FP的位置来推算伪SP的位置，这里之前我是比较困惑的；我们在《汇编语言》这本书中知道函数调用的时候是需要EBP和ESP寄存器的；**但是**，其实这并不是所有汇编语言的特点，这个应该算是X86的calling convention，这其实是一套约定促成的方法：
> 在IA32上，通常每个函数调用的时候都会分配一个栈帧，栈帧内保存了一些数据，比如参数，局部变量之类的。ebp和esp就指向栈帧的一头一尾。由于esp会经常改变，所以ebp作为stack frame base pointer，这样定位栈帧中的数据会比较方便。当然，这也只是通常，随着calling conversion的改变而改变。比如说X86_64通常就只用到了rsp而不再需要帧指针，而且IA32上也能通过frame pointer omission optimization把帧指针优化掉。
上述观点详见[汇编中为什么需要帧指针%ebp?](https://www.zhihu.com/question/284579060)。

因此，我们首先需要知道：**在函数调用中，没有使用bp寄存器**是完全可以的，并且由于在go 1.17版本之前，函数调用之前会预先分配出需要使用的栈空间，因此plan9中的SP其实是一开始就分配好的，那么真实的BP寄存器其实就不是必须的了，对于像我这种之前只是了解IA32的同学，可能会有点强迫症，所以没有BP寄存器完全可以。
另外在gosave这个保存goroutine现场的代码中：
```
TEXT gosave(SB), 7, $0
	MOVQ	8(SP), AX		// gobuf
	MOVQ	SP, 0(AX)		// save SP
	MOVQ	0(SP), BX
	MOVQ	BX, 8(AX)		// save PC
	MOVL	$0, AX			// return 0
	RET
```
我们可以看到，只需要PC和SP，**BP并不是必须的**。
并且在曹大的文章中，有说把framepointer压栈的场景：
> 此外需要注意的是，caller BP 是在编译期由编译器插入的，用户手写代码时，计算 frame size 时是不包括这个 caller BP 部分的。是否插入 caller BP 的主要判断依据是:
> 1. 函数的栈帧大小大于 0
> 2. 下述函数返回 true
> func Framepointer_enabled(goos, goarch string) bool {
>    return framepointer_enabled != 0 && goarch == "amd64" && goos != "nacl"
> }
那就是说伪FP和伪SP有时会只隔了一个return address，有时候又隔了return address和framepointer.

另外还需要注意的一个点是：
FP和Go 的官方源代码里的framepointer不是一回事，源代码里的framepointer指的是caller BP寄存器的值(x86的习惯)。

我们知道函数调用时，会插入return address，这部分空间会计算在caller stack中；另外可能会在callee中插入caller BP，这部分空间是计算在callee中的；
**但是**，在我们手写plan9汇编的时候，需要有一些注意的地方：
* 在函数中是否有对其它函数调用时，如果有的话，调用时需要将callee的参数、返回值考虑在内。虽然 return address(rip)的值也是存储在 caller 的 stack frame 上的，但是这个过程是由 CALL 指令和 RET 指令完成 PC 寄存器的保存和恢复的，在**手写汇编**时，同样也是不需要考虑这个PC寄存器在栈上所需占用的 8 个字节的(因此可以认为硬件SP就是指向了除return address外的第一个数据起始位置，硬件SP请看文章的第一个示意图)。
* 如果是offset(SP)则表示硬件寄存器SP。务必注意。对于编译输出(go tool compile -S / go tool objdump)的代码来讲，目前所有的SP都是硬件寄存器SP，无论是否带symbol。
* caller BP 是在编译期由编译器插入的，用户手写代码时，计算frame size时是不包括这个caller BP部分的。

最后以一个函数调用全景图完成这篇概要总结分析：
```
                                                                                                                              
                                       caller                                                                                 
                                 +------------------+                                                                         
                                 |                  |                                                                         
       +---------------------->  --------------------                                                                         
       |                         |                  |                                                                         
       |                         | caller parent BP |                                                                         
       |           BP(pseudo SP) --------------------                                                                         
       |                         |                  |                                                                         
       |                         |   Local Var0     |                                                                         
       |                         --------------------                                                                         
       |                         |                  |                                                                         
       |                         |   .......        |                                                                         
       |                         --------------------                                                                         
       |                         |                  |                                                                         
       |                         |   Local VarN     |                                                                         
                                 --------------------                                                                         
 caller stack frame              |                  |                                                                         
                                 |   callee arg2    |                                                                         
       |                         |------------------|                                                                         
       |                         |                  |                                                                         
       |                         |   callee arg1    |                                                                         
       |                         |------------------|                                                                         
       |                         |                  |                                                                         
       |                         |   callee arg0    |                                                                         
       |                         ----------------------------------------------+   FP(virtual register)                       
       |                         |                  |                          |                                              
       |                         |   return addr    |  parent return address   |                                              
       +---------------------->  +------------------+---------------------------    <-------------------------------+         
                                                    |  caller BP               |                                    |         
                                                    |  (caller frame pointer)  |                                    |         
                                     BP(pseudo SP)  ----------------------------                                    |         
                                                    |                          |                                    |         
                                                    |     Local Var0           |                                    |         
                                                    ----------------------------                                    |         
                                                    |                          |                                              
                                                    |     Local Var1           |                                              
                                                    ----------------------------                            callee stack frame
                                                    |                          |                                              
                                                    |       .....              |                                              
                                                    ----------------------------                                    |         
                                                    |                          |                                    |         
                                                    |     Local VarN           |                                    |         
                                  SP(Real Register) ----------------------------                                    |         
                                                    |                          |                                    |         
                                                    |                          |                                    |         
                                                    |                          |                                    |         
                                                    |                          |                                    |         
                                                    |                          |                                    |         
                                                    +--------------------------+    <-------------------------------+         
                                                                                                                              
                                                              callee
```
