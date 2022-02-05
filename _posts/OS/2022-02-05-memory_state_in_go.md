---
layout: article
title: Linux中的Go地址空间状态转换函数浅析
comments: true
---


> 本文将对Go语言中内存状态的部分细节进行阐述，欢迎一起讨论指正。

# Linux中的Go地址空间状态转换函数浅析

本文主要是简要地介绍Go源码中抽象出来的**地址空间状态**在转换时涉及到的各个函数细节，以便更加清晰地理解*None*，*Reserved*，*Prepared*，*Ready*这四个状态真正的意义。需要说明的是，本文以**draveness**大佬的[7.1 内存分配器](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/)作为前提。

## 一、Go地址空间背景知识

下图是地址空间状态转换图，取自[7.1 内存分配器](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/)一文；

![地址空间的状态转换](https://img.draveness.me/2020-02-29-15829868066474-memory-regions-states-and-transitions.png)

图中可以看到有以下几个函数：

* sysAlloc
* sysFree
* sysReserve
* sysMap
* sysUsed
* sysUnused
* sysFault

再次引用**draveness**关于这四个状态的表格：

|    状态    |                             解释                             |
| :--------: | :----------------------------------------------------------: |
|   `None`   |         内存没有被保留或者映射，是地址空间的默认状态         |
| `Reserved` |        运行时持有该地址空间，但是访问该内存会导致错误        |
| `Prepared` | 内存被保留，一般没有对应的物理内存访问该片内存的行为是未定义的可以快速转换到 `Ready` 状态 |
|  `Ready`   |                        可以被安全访问                        |

## 二、Linux中的系统调用

我们以1.14.12中的代码来做说明，各个系统调用函数具体细节如下：

* sysAlloc

  ```
  func sysAlloc(n uintptr, sysStat *uint64) unsafe.Pointer {
  	// 主要关注mmap中的prot字段和flag字段
  	// prot:映射区域的保护方式。PROT_READ——映射区域可被读取，PROT_WRITE——映射区域可被写入
  	// flag:影响映射区域的各种特性。在调用mmap()时必须要指定MAP_SHARED 或MAP_PRIVATE
  	//      MAP_ANON建立匿名映射。此时会忽略参数fd，不涉及文件，而且映射区域无法和其他进程共享
  	//      该特性会映射出一块0值的内存区域;
  	//      MAP_PRIVATE 对映射区域的写入操作会产生一个映射文件的复制，即私人的“写入时复制”
  	//      （copy on write）对此区域作的任何修改都不会写回原来的文件内容
  	p, err := mmap(nil, n, _PROT_READ|_PROT_WRITE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
  	if err != 0 {
  		if err == _EACCES {
  			print("runtime: mmap: access denied\n")
  			exit(2)
  		}
  		if err == _EAGAIN {
  			print("runtime: mmap: too much locked memory (check 'ulimit -l').\n")
  			exit(2)
  		}
  		return nil
  	}
  	mSysStatInc(sysStat, n)
  	return p
  }
  ```

  从上面的源代码中可以看出`sysAlloc`函数创建出了一个给程序自己使用的初始化过的内存区域。

* sysUnused

  相关代码如下：

  ```
  func sysUnused(v unsafe.Pointer, n uintptr) {
  	// By default, Linux's "transparent huge page" support will
  	// merge pages into a huge page if there's even a single
  	// present regular page, undoing the effects of madvise(adviseUnused)
  	// below. On amd64, that means khugepaged can turn a single
  	// 4KB page to 2MB, bloating the process's RSS by as much as
  	// 512X. (See issue #8832 and Linux kernel bug
  	// https://bugzilla.kernel.org/show_bug.cgi?id=93111)
  	//
  	// To work around this, we explicitly disable transparent huge
  	// pages when we release pages of the heap. However, we have
  	// to do this carefully because changing this flag tends to
  	// split the VMA (memory mapping) containing v in to three
  	// VMAs in order to track the different values of the
  	// MADV_NOHUGEPAGE flag in the different regions. There's a
  	// default limit of 65530 VMAs per address space (sysctl
  	// vm.max_map_count), so we must be careful not to create too
  	// many VMAs (see issue #12233).
  	//
  	// Since huge pages are huge, there's little use in adjusting
  	// the MADV_NOHUGEPAGE flag on a fine granularity, so we avoid
  	// exploding the number of VMAs by only adjusting the
  	// MADV_NOHUGEPAGE flag on a large granularity. This still
  	// gets most of the benefit of huge pages while keeping the
  	// number of VMAs under control. With hugePageSize = 2MB, even
  	// a pessimal heap can reach 128GB before running out of VMAs.
  	if physHugePageSize != 0 {
  		// If it's a large allocation, we want to leave huge
  		// pages enabled. Hence, we only adjust the huge page
  		// flag on the huge pages containing v and v+n-1, and
  		// only if those aren't aligned.
  		var head, tail uintptr
  		if uintptr(v)&(physHugePageSize-1) != 0 {
  			// Compute huge page containing v.
  			head = alignDown(uintptr(v), physHugePageSize)
  		}
  		if (uintptr(v)+n)&(physHugePageSize-1) != 0 {
  			// Compute huge page containing v+n-1.
  			tail = alignDown(uintptr(v)+n-1, physHugePageSize)
  		}
  
  		// Note that madvise will return EINVAL if the flag is
  		// already set, which is quite likely. We ignore
  		// errors.
  		if head != 0 && head+physHugePageSize == tail {
  			// head and tail are different but adjacent,
  			// so do this in one call.
  			madvise(unsafe.Pointer(head), 2*physHugePageSize, _MADV_NOHUGEPAGE)
  		} else {
  			// Advise the huge pages containing v and v+n-1.
  			if head != 0 {
  				madvise(unsafe.Pointer(head), physHugePageSize, _MADV_NOHUGEPAGE)
  			}
  			if tail != 0 && tail != head {
  				madvise(unsafe.Pointer(tail), physHugePageSize, _MADV_NOHUGEPAGE)
  			}
  		}
  	}
  
  	if uintptr(v)&(physPageSize-1) != 0 || n&(physPageSize-1) != 0 {
  		// madvise will round this to any physical page
  		// *covered* by this range, so an unaligned madvise
  		// will release more memory than intended.
  		throw("unaligned sysUnused")
  	}
  
  	var advise uint32
  	// 在Linux 4.5之前，都是使用MADV_DONTNEED这个参数，从4.5开始，释放内存时是使用MADV_FREE参数
  	// 这两个参数都是释放内存，但是有一些比较细节的地方，例如虽然我们调用了madvise函数，但是内核并不是
  	// 马上就释放了该块内存等，具体细节可以查看https://man7.org/linux/man-pages/man2/madvise.2.html
  	// 需要说明的是这里的释放是虚拟地址空间对应的物理内存，即告诉内核这一片地址对应的物理内存可以回收，如果程序再想使用该
  	// 虚拟空间时但是对应的物理空间已经被内核拿去它用，则会让内核按需重新分配物理内存
  	if debug.madvdontneed != 0 {
  		advise = _MADV_DONTNEED
  	} else {
  		advise = atomic.Load(&adviseUnused)
  	}
  	if errno := madvise(v, n, int32(advise)); advise == _MADV_FREE && errno != 0 {
  		// MADV_FREE was added in Linux 4.5. Fall back to MADV_DONTNEED if it is
  		// not supported.
  		atomic.Store(&adviseUnused, _MADV_DONTNEED)
  		madvise(v, n, _MADV_DONTNEED)
  	}
  }
  ```

* sysUsed

  ```
  func sysUsed(v unsafe.Pointer, n uintptr) {
  	// Partially undo the NOHUGEPAGE marks from sysUnused
  	// for whole huge pages between v and v+n. This may
  	// leave huge pages off at the end points v and v+n
  	// even though allocations may cover these entire huge
  	// pages. We could detect this and undo NOHUGEPAGE on
  	// the end points as well, but it's probably not worth
  	// the cost because when neighboring allocations are
  	// freed sysUnused will just set NOHUGEPAGE again.
  	sysHugePage(v, n)
  }
  
  func sysHugePage(v unsafe.Pointer, n uintptr) {
  	if physHugePageSize != 0 {
  		// Round v up to a huge page boundary.
  		beg := alignUp(uintptr(v), physHugePageSize)
  		// Round v+n down to a huge page boundary.
  		end := alignDown(uintptr(v)+n, physHugePageSize)
  
  		if beg < end {
  		  // 该函数是sysUsed的关键函数，主要是使用了MADV_HUGEPAGE参数，该参数主要是对v unsafe.Pointer开始的内存使用THP
  		  // 对MADV_HUGEPAGE这个的相关知识点如下(在[https://www.onitroad.com/jc/linux/man-pages/linux/man2/madvise.2.html]中了解到的)：
  		  // 在地址和长度指定的范围内的页面上启用透明大页面(THP——Transparent Huge Page)。当前，"透明大页面"仅与私有匿名页面
  		  // 一起使用(请参阅mmap(2))。内核将定期扫描标记为大页面候选者的区域，以将其替换为大页面。当区域自然地与大页面大小对齐时，
  		  // 内核还将直接分配大页面(请参阅posix_memalign(2))。
        // 此功能主要针对使用大型数据映射并一次访问该内存较大区域的应用程序(例如，虚拟化系统，例如QEMU)。它很容易浪费内存
        // (例如，仅访问1个字节的2 MB映射将导致2 MB的有线内存，而不是一个4 KB的页面)。有关更多详细信息，请参见Linux内核源文件
        // Documentation / admin-guide / mm / transhuge.rst。
        // 默认情况下，大多数常见的内核配置都提供MADV_HUGEPAGE样式的行为，因此，通常不需要MADV_HUGEPAGE。
        // 它主要用于嵌入式系统，其中默认情况下可能未在内核中启用MADV_HUGEPAGE样式的行为。在此类系统上，可以使用此标志来选择
        // 性地启用THP。每当使用MADV_HUGEPAGE时，它都应该始终位于具有开发模式事先知道不会启用增加的透明大页的应用程序的内存
        // 占用的风险的访问模式的内存区域中。
        // 仅当内核配置了CONFIG_TRANSPARENT_HUGEPAGE时，MADV_HUGEPAGE和MADV_NOHUGEPAGE操作才可用。
  			madvise(unsafe.Pointer(beg), end-beg, _MADV_HUGEPAGE)
  		}
  	}
  }
  
* sysFree

  ```
  func sysFree(v unsafe.Pointer, n uintptr, sysStat *uint64) {
  	mSysStatDec(sysStat, n)
  	munmap(v, n)
  }
  ```

  需要说明上面的munmap，是go语言根据操作系统及cpu架构自动生成的系统调用代码，相关知识点可以查看https://go.dev/src/cmd/vendor/golang.org/x/sys/unix/README 中关于系统调用函数生成的方式；简单来说，就是使用mksyscall.pl脚本根据具体的操作系统和CPU架构生成相应的系统调用文件及其中相关的调用函数，例如go语言会生成linux系统、amd架构，是使用如下的指令：

  > mksyscall.pl -tags linux,amd64 syscall_linux.go syscall_linux_amd64.go

  使用脚本生成的一般都是比较简单的，一半都是系统调用ID+函数参数

* sysFault

  ```
  func sysFault(v unsafe.Pointer, n uintptr) {
    // 我们已经在前文中的sysAlloc有说明mmap中的MAP_ANON参数和MAP_PRIVATE，针对
    // MAP_FIXED参数，可以这样理解：如果参数v所指的地址无法成功建立映射时，则放弃映射，不对地址做修正。
  	mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE|_MAP_FIXED, -1, 0)
  }
  ```
  
* sysReserve

  ```
  func sysReserve(v unsafe.Pointer, n uintptr) unsafe.Pointer {
  	// mmap中的参数中的其他参数在上文中已经表述过了，该函数中的mmap中的第三个参数prot被设置成了PROT_NONE
  	// PROT_NONE表示映射区不能存取，这样以v作为开始地址，长度为n的内存区域就不能使用了，这样被保留下来了
  	p, err := mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
  	if err != 0 {
  		return nil
  	}
  	return p
  }
  ```
  
* sysMap

  ```
  func sysMap(v unsafe.Pointer, n uintptr, sysStat *uint64) {
  	mSysStatInc(sysStat, n)
  
  	// sysMap中使用mmap来映射之前已经分配好的起始地址为v，长度为n的内存区域；
  	p, err := mmap(v, n, _PROT_READ|_PROT_WRITE, _MAP_ANON|_MAP_FIXED|_MAP_PRIVATE, -1, 0)
  	if err == _ENOMEM {
  		throw("runtime: out of memory")
  	}
  	if p != v || err != 0 {
  		throw("runtime: cannot map pages in arena address space")
  	}
  }
  ```

通对`mem_linux.go`中各个关键函数分析之后，我们再看文章开始的那个图就会有更深层次的理解，首先解释一下各个状态：

1. None：默认初始状态，例如准备开始申请内存区域（调用mmap申请内存）、或者已经申请的内存区域不再使用（调用munmap释放内存）的状态；
2. Reserved：对拥有的内存区域进行保留，在`mem_linux.go`中就是将那块内存设置成PROT_NONE的状态，即拥有该内存区域但是目前无法使用的状态；
3. Prepared：即该内存已经是分配出来的状态，但是目前进程还不急着使用，所以告诉内核该块虚拟内存区域对应的物理内存可以由内核暂时拿去使用，等我要使用该虚拟内存时再告知内核去重新获取物理内存即可；
4. Ready：就是对于进行来说已经分配好的内存区域，可以直接进行使用的；

现在对逐个状态的转换进行解释：

1. None--[sysAlloc]--->Ready：通过使用sysAlloc来调用mmap分配出可读可写的内存区域并返回虚拟地址空间的起始 地址；

2. None--[sysReserve]-->Ready：通过使用sysReserve来分配出PROT_NONE属性的内存地址，留着后续进行直接mmap成PROT_READ ｜PROT_WRITE属性的内存区域；

3. Reserved--[sysFree]-->None：通过使用sysFree来调用munmap系统调用直接remove the mapping，即将之前映射出来的内存区域直接移除；
4. Prepared--[sysFree]-->None：通过使用sysFree来munmap**之前使用advise将内存区域设置成MADV_DONTNEED或者MADV_FREE的内存区域**直接remove掉，该块虚拟地址空间也变成了None状态；
5. Ready--[sysFree]-->None：和第3、4点同理，不再解释，就是初始状态不同；
6. Reserved--[sysMap]-->Prepared：通过使用sysMap来mmap之前已经**被mmap为PROT_NONE属性的内存区域**，将其重新mmap成PROT_READ｜PROT_WRITE属性的内存区域；
7. Prepared--[sysFault]-->Reserved：通过使用sysFault来mmap之前被madvise为**MADV_DONTNEED或者MADV_FREE的内存区域**充值设置成PROT_NONE 状态的内存块，如果该块虚拟内存对应的物理内存已经被内核拿去他用的话，就会由内核再次为该虚拟内存区域重新分配物理内存；
8. Prepared--[sysUsed]-->Ready：即对之前被madvise设置为MADV_DONTNEED或者MADV_FREE的内存区域的再重新设置成MADV_HUGEPAGE进行后续的使用；
9. Ready--[sysUnused]-->Prepared：和第8点是反过来的，不再赘述；
10.  Ready--[sysFault]-->Reserved：将已经分配好的内存通过sysFault函数利用mmap设置成PROT_NONE属性的内存区域；

## 三、总结

对所有的状态及状态转换的解释目前全部完成，我自己也对这几个状态有了更深的理解；同时也感叹大牛们对内存抽象能力的敬佩；将各个操作系统的内存管理进行高度统一，十分敬佩。

另外，如果文章中有误人子弟的地方，请各位读者留言指出，我会及时修改。

