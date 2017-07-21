---
layout: article
title: 逻辑地址、线性地址与虚拟地址的联系
comments: true
---


> 本文将简单地分析一下逻辑地址，线性地址和虚拟地址之间的区别


（以下解释中一部分是我自己稍微修改了一下，一部分是直接引用别人在在知乎留下的答案）

------

#三种地址之间的区别

在intel的平台下，逻辑地址是selector:offset这种形式来表示的，其中selector（我们翻译为段选择子）是CS、DS、SS等段寄存器的高13bit的值，剩下的3bit分别用来描述优先级等信息；offset是段内偏移地址；cpu通过段寄存器中段选择子的内容（这个内容其实就是global description table中的index）去全局描述符表中找到其对应的base address（基址），通过和offset（段内偏移地址）相加得到linear address（线性地址），我们把这个过程称为段式内存管理。

获取到linear address之后，CPU会继续将linear address分成四段，用前三段分别做索引去PGD、PMD、Page Table中去查表项（Page Table Entry），该表项的值就是某个物理内存页的首地址；此时再使用linear address第四段的值作为页内偏移地址就可以得到最后的物理地址，我们把这个过程称为页式内存管理。

问题来了，为什么没提到 virtual address，这是个什么东西？其实在 Intel IA-32 手册里并没有提到这个术语，但是在内核的确是用到了这个概念，比如__va和__pa这两个宏定义。看似神秘的 virtual address 究其本质就是程序里面使用的地址比如一个指针值，指针的本质就是 EIP 寄存器里的值，说直白点，virtual address 就是 EIP 寄存器的值。你会发现我们上面说过，logical address 由 selector 和 offset 两部分组成，offset 也是 EIP 寄存器的值，所以结论为：logical address 的 offset 正是 virtual address，它俩是一个东西。

既然搞明白了 logical address 和 virtual address 的关系，那么我们再来看下，linear address 和 virtual address 是什么关系。在上面讲到的段式内存管理中，Linux 内核会将 segment base address(段基址)设成 0，于是就有 linear address = 0+offset，又因为 virtual address 就是 offset，所以算出的 linear address在数值上等于 virtual address，注意，是数值上等于，它们之间是差了段基址的，只不过段基址为 0 罢了。

#随便说说

网上很多资料认为逻辑地址是虚拟地址的别名，其实它们不是一个东西。还有很多资料把线性地址当作虚拟地址的别名，其实它们也不是一个东西，只是Linux在x86下将它们搞得数值相等而已，虽然值相等但是本质不同。

讲到这里，三者之间的关系就讲明白了，最后说下为什么这三个概念会如此混乱。
按照 Intel 的设计，段式内存管理中的段类型分为三种：代码段(上面讲了)、数据段、系统段(TSS之类的)，实在是太麻烦了（这是大家包括ucore中也不想使用段式内存管理的原因）。我们只靠页式内存管理就已经可以完成Linux内核需要的所有功能，根本不需要段映射，段映射的功能完全可以依靠页式内存管理来完成；但是段映射在boot加载的过程中就需要开启（建立全局描述符表），那就只能上点手段了。于是，Linux内核将所有类型的段的 segment base address 都设成0，段限长都设成最大，那么这样一来所有段都重合了，也就是不分段了；此外由于段限长是地址总线的寻址限度，所以这也相当于所有段跟整个线性空间重合了。虚拟地址本来是在段内的偏移量，现在段就是整个线性空间，所以虚拟地址就成了在整个线性空间内的偏移量，这和线性地址的概念一样，所以内核开发者都已经将虚拟地址和线性地址当作一个东西了。像是 Understand The Linux Kernel 这本书里面为了避免混淆，除了在开头和术语表中引用了 virtual address 这个词组之外，其他地方全是用的 linear address。

看完这个答案，你会发现我们绕了一圈回来，虽然逻辑地址的概念很清晰，但是虚拟地址和线性地址依然可以不作区分，因为区分了也没什么用，内核里这俩概念是通用的。不过，知道点区别还是不至于在某些时候把自己搞晕，尤其是有些书和教程里面这两个词不说缘由就混着用，这很蛋疼。好了，最后结论就是，这两个概念区分开来的确更加清晰，但如果不作区分而直接把虚拟地址看作线性地址的别名，对你理解内核也不会产生任何影响。

（注：在此对知乎用户[HAO LEE](https://www.zhihu.com/people/kerneldevelope) 表示感谢）
