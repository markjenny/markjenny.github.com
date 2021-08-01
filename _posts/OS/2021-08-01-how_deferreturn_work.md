# deferrenturn如何实现循环处理

我们知道当程序使用了defer关键字后，会在调用defer处注册defer函数，并在函数RET前执行deferreturn函数；那一个deferreturn函数如何把所有的defer函数全部执行完呢？
我们可以首先看一下deferreturn函数：
```go
// Run a deferred function if there is one.
// The compiler inserts a call to this at the end of any
// function which calls defer.
// If there is a deferred function, this will call runtime·jmpdefer,
// which will jump to the deferred function such that it appears
// to have been called by the caller of deferreturn at the point
// just before deferreturn was called. The effect is that deferreturn
// is called again and again until there are no more deferred functions.
// Cannot split the stack because we reuse the caller's frame to
// call the deferred function.
// The single argument isn't actually used - it just has its address
// taken so it can be matched against pending defers.
//go:nosplit
func deferreturn(arg0 uintptr) { // arg0是defer函数的参数位置，这个位置应该是caller函数的堆栈内
    gp := getg()
    d := gp._defer
    if d == nil {
        return
    }
    sp := getcallersp()
    if d.sp != sp {
        return
    }
    // Moving arguments around.
    //
    // Everything called after this point must be recursively
    // nosplit because the garbage collector won't know the form
    // of the arguments until the jmpdefer can flip the PC over to
    // fn.
    switch d.siz {
    case 0:
        // Do nothing.
    case sys.PtrSize:
        *(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
    default:
        memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz)) // 将_defer结构中保存的参数再拷贝回caller队长内
    }
    fn := d.fn
    d.fn = nil
    gp._defer = d.link // 去掉该链表节点
    freedefer(d) // 清空_defer结构，并将该结构存储回P的deferpool中，注意deferpool是属于P的
    jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}
```
如果从deferreturn这个函数的角度来看，这是里并没有在函数调用上有什么特殊处理，秘密都来自jmpdefer函数，下面列出jmpdefer函数代码：
```asm
// func jmpdefer(fv *funcval, argp uintptr)
// argp is a caller SP.
// called from deferreturn.
// 1. pop the caller
// 2. sub 5 bytes from the callers return
// 3. jmp to the argument
TEXT runtime·jmpdefer(SB), NOSPLIT, $0-16
	MOVQ	fv+0(FP), DX	// fn，FP是位寄存器，指向calller传递给callee的第一个参数，因此这个参数目前就是funcval指针
	MOVQ	argp+8(FP), BX	// caller sp；我们知道这个argp其实是deferreturn函数参数
	LEAQ	-8(BX), SP	// caller sp after CALL; 而又因为我们是在模拟调用jmpdefer，所以还需要有return address，因此这里将BX-8作为调用后的SP； 取(BX-8)地址所指向的值再取地址作为SP，相当于是BX值减去8赋值给SP，那就是caller的SP
	MOVQ	-8(SP), BP	// restore BP as if deferreturn returned (harmless if framepointers not in use)；这里是计算caller的BP值，相当于是将SP-8处的值赋值给BP，相当于是复原了caller的BP
	SUBQ	$5, (SP)	// return to CALL again; 
	MOVQ	0(DX), BX 取fucnval的值给BX寄存器，funcval中的第一参数就是函数地址值
	JMP	BX	// but first run the deferred function; 调用被defer的函数
```
为了后续更好描述，我们首先给几个函数定义一下专有名词：使用了defer关键字的函数是`caller`，被defer执行的函数是`deferred func`;
下面开始解读函数jmpdefer:
在deferreturn函数中的jmpdefer函数有两个参数fn和uintptr(unsafe.Pointer(&arg0))，第一个参数是封装了deferred func的funcval结构体指针，第二个参数是deferred func所要使用的第一个参数的指针；
`MOVQ	fv+0(FP), DX`将第一个参数的地址赋给了BX寄存器；如果要实现**caller直接调用deferred func**的效果，那么CALL指令还会在caller的堆栈中，所以`LEAQ	-8(BX), SP`相当于是重置SP将其往下移8个字节从而把return address的位置留出来；
`MOVQ	-8(SP), BP`这个命令相当于恢复caller的BP；`-8(SP)`其实是deferred func被调用时的BP，这个BP存储的就是caller的BP，所以`MOVQ	-8(SP), BP`相当于是**读取【SP指针的值再减8后】的地址**并赋值给BP，这相当于是`POP BP`的操作了，该指令执行过后，BP、SP寄存器都恢复成了caller自己的BP和SP；
下面这一句是deferreturn可以循环执行的关键，原谅我之前的才疏学浅这里之前怎么也理解不了，多亏拜读了该文章————[defer 链如何被遍历](https://www.cnblogs.com/qcrao-2018/p/12550380.html)后，醍醐灌顶；
`SUBQ $5, (SP)`把返回地址减少了 5B，刚好是一个 CALL 指令的长度(**我们前文说过SP当前指向的位置是caller调用deferreturn函数后的下一条指令**，相当于是又将return address指向了指令`CALL deferreturn`这个指令，那么在调用完deferred func后还会再次执行deferreturn函数)。

引用一下上述文章中的段落：
> 什么意思？当执行完 deferreturn 函数之后，执行流程会返回到 CALL deferreturn 的下一条指令，将这个值减少 5B，也就又回到了 CALL deferreturn 指令，从而实现了“递归地”调用 deferreturn 函数的效果。当然，栈却不会在增长！

下面两条语句比较简单：`MOVQ	0(DX), BX`解引用`*funcval`值的并赋值给BX，通过查看funcval结构可知现在BX就是存储了函数地址:
```go
type funcval struct {
	fn uintptr
	// variable-size, fn-specific data here
}
```
`JMP	BX`跳转到相应的函数地址进行执行，`jmpdefer`函数完成了caller调用deferred func的功能，个人觉得整个过程真实精彩！

