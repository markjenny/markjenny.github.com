---
layout: article
title: mov指令和lea指令对比
comments: true
---


> 本文将简单地分析movq指令和leaq指令对比之间的区别

------

## mov指令
mov就是做直接复制操作；
例如：
* movq %ebx, %eax； 就是将ebx寄存器中的值赋值给eax;
* movq (%ebx), %eax；就是将ebx中存储的地址对应的值赋值给eax，这里需要注意的是，一旦寄存器外围使用了()这种括号操作，相当于是解引用——读取寄存器中的值所指向内存对应的值；

## lea指令
lea（Load Effective Address [Quad])即将一个内存地址直接赋值给目的操作数
`leaq S, D            (D <- &S)`
我们可以举一个例子：
`leaq %eax, (%ebx+8)`就是将`ebx+8`这个值直接复制给`eax`，而不是把`ebx+8`处的内存地址里的数据赋值给`eax`。
上面最后一条指令其实是最迷惑的，我们可以尝试这样进行理解：
首先`(%ebx+8)`就是解引用操作，相当于是读取`ebx+8`地址处的数值，然后`lea`又是读取相关值的地址，那么就相当于是:

```
int *ebx = 23333; // 纯是为了理解。。。
eax = &(*ebx); // 那么这个值其实还是23333
```

因此lea指令就可以做一些算数运算

```
long m12(long x) {
    return x*12
}
```

相关的汇编指令如下：

```
leaq (%rdi, %rdi, 2), %rax # t <- x+x*2
salq $2, %rax # return t<<2或者，
```

`# %rax = x    %rcx = y`

| Expression | result in rdx |
|  ----  | ----  |
| leaq 6(%rax)， %rdx | x+6 |
| leaq (%rax, %rcx), %rdx | x+y |
| leaq (%rax, %rcx, 4), %rdx | x+4y |
| leaq 7(%rax, %rax, 8) | 9x+7 |
| leaq 0XA(, %rcx, 4), %rdx | 4y+10 |
| leaq 9(%rax, %rcx, 2), %rdx | x+2y+9 |


参照的AT&T风格指令，旨在理解lea的指令意义，如果有问题请指正（去issue下面提即可，评论暂时还没用弄==）。
