## 1. 前言
在很多高级语言中，都有多线程的实现，所谓的多线程指的就是通过分时技术，线程不断切换运行，达到多个线程近似同时运行的效果。现在很多网站都有很高的并发，而高并发的基础，就是操作系统对于多进程多线程的调度与切换的优秀实现，本文就基于linux0.11版本，分析linux进程切换函数switch_to的实现。

## 2. 汇编
先来看一段汇编代码:

```c
__asm__ __volatile__ ( 
LOCK "addl %1, %0" 
: "=m" (v->counter) 
: "ir" (i), "m" (v->counter));
```
这是一段asm嵌入式汇编，一般来说，asm嵌入式汇编分四个部分，每个部分以 ":"隔开，这是基本的格式:

指令部 : 输出部 ：输入部 ：损坏部

根据这个格式，可以对上面的代码做一个划分:

指令部 ： "addl %1, %0" 
输出部 ： "=m" (v->counter) 
输入部 ： "ir" (i), "m" (v->counter))
损坏部： 无

首先看指令部，addl指令是加法指令，而%+数字这种格式，这些数字代表着一个编号，也就是cpu的通用寄存器的编号，这个编号的由来是，从输入部开始算，每个操作数（变量）依次编号，第一个操作数对应%0，第二个对应%1。

这么说可能有点难理解，结合代码来说就是，输入部第一个操作数是"ir" (i),因此它就对应%0，第二个操作数是"m" (v->counter))，因此它就对应%1，依次类推，这个数字能用的最大值就是cpu的通用寄存器个数。

另外我们可以看到，每个操作数前会有“m”,"r"这种的字母前缀，这是汇编的约束，约束就是规定了操作数的存储地址，表示约束的字母有

```c
"m","v","o"                     ——表示内存单元；
"r"                             ——表示任何寄存器；
"q"                             ——表示eax,ebx,ecs,edx 之一；
"i"和"h"                        ——表示直接操作数；
"E"和"F"                        ——表示浮点数；
"g"                             ——表示“任意”；
"a"，"b"，"c"，"d"               ——表示使用寄存器eax,ebx,ecx,edx
"S"和"D"                        ——表示要求使用寄存器esi和edi
"I"                             ——表示常数（0至31）
```
所以对应我们的事例代码，就可以看懂了。%0对应输入部第一个参数i，操作数i是一个直接操作数，%1对应v->counter，这是内存单元中的一个操作数，指令addl将二者相加，相加的结果输出到输出部，而输出部就是内存单元中的v->counter。

简单来说就是将一个内存中的操作数v->counter加上一个直接操作数i，结果再输出到v->count。

## 3. switch_to

```c
#define switch_to(n) {\
struct {long a,b;} __tmp; \
__asm__(              \
    "cmpl %%ecx,current\n\t" \
	"je 1f\n\t" \
	"movw %%dx,%1\n\t" \
	"xchgl %%ecx,current\n\t" \
	"ljmp *%0\n\t" \
	"cmpl %%ecx,last_task_used_math\n\t" \
	"jne 1f\n\t" \
	"clts\n" \
	"1:" \
	::"m" (*&__tmp.a),"m" (*&__tmp.b), \
	"d" (_TSS(n)),"c" ((long) task[n])); \
}
```

这是switch_to的源码，它存在于kernel的sched.h头文件中。

首先定义了一个结构体_tmp,_tmp中只有两个long型变量a,b，这两个变量后面会用到。

根据第二节的汇编语言的结构，可以对switch_to的源码做以下分解:

指令部:

```c
	"cmpl %%ecx,current\n\t" \
	"je 1f\n\t" \
	"movw %%dx,%1\n\t" \
	"xchgl %%ecx,current\n\t" \
	"ljmp *%0\n\t" \
	"cmpl %%ecx,last_task_used_math\n\t" \
	"jne 1f\n\t" \
	"clts\n" \
	"1:" \
```
输出部: 无
输入部: `"m" (*&__tmp.a),"m" (*&__tmp.b), \"d" (_TSS(n)),"c" ((long) task[n]))`


同时我们也可以对指令部用到的参数和输入部做一个匹配:

```bash
%ecx	 —————— ecx寄存器，也就是输入的task[n]，也就是要切换到的新进程
%dx		 —————— edx寄存器，也就是输入的_TSS(n)
%1		 —————— "m" (*&__tmp.b)
%0		 —————— "m" (*&__tmp.a)
```

有了这些准备我们就可以来逐步分析switch_to的代码了。

指令部首先执行了这一句

```bash
"cmpl %%ecx,current\n\t" \
```
cmpl指令是比较指令，current是一个代表当前正在运行的进程的结构体变量task_struct，这个task_struct我们之前提到过，它代表着一个进程的一些属性，如进程id，进程状态等等。这条指令就是将当前正在运行的进程和要切换过去运行的进程做一个比较，如果二者是相同的，则下一条指令`je 1f\n\t" `会跳转到标号为1的地方执行，而标号为1的地方是`"1:" \`，也就是什么都不做。

这个容易理解，如果要切换过去的进程和当前进程是同一个进程，那就不用切换睐，等着重新调度吧。

接着下一条指令`"movw %%dx,%1\n\t" \`,movw指令是一条传送指令，就是将一个操作数传送到目的地址。因此这条指令的含义就是将_TSS(n)的值赋给_tmp.b。

这个_TSS(n)大有来头，扮演了非常重要的角色。我们都知道TSS是进程的状态段，它用来保存进程切换的现场，以便于进程的恢复。_TSS的定义

```c
/*
 * Entry into gdt where to find first TSS. 0-nul, 1-cs, 2-ds, 3-syscall
 * 4-TSS0, 5-LDT0, 6-TSS1 etc ...
 */
#define FIRST_TSS_ENTRY 4
#define _TSS(n) ((((unsigned long) n)<<4)+(FIRST_TSS_ENTRY<<3))
```

为了更好的理解_TSS(n)，先介绍一下全局描述符GDT的结构。上文中我们介绍过，Linux用一个数组GDT来保存全局的一些地址，如进程的LDT段，TSS段等等，它的构成如下：
![GDT结构](https://img-blog.csdnimg.cn/0ce11a3c0b2c43bfb2bb45aab96f0e4c.png)
从图中可以看出，第一个tss段的索引是4，对应这FIRST_TSS_ENTRY，而tss段的结构为:
![在这里插入图片描述](https://img-blog.csdnimg.cn/c7b07b0330f14dffba2517abd542524e.png)
前三位无用，因此FIRST_TSS_ENTRY左移3作为基地址，加上n作为偏移地址找到切换的进程的tss段。

因此_TSS(n)就代表了进程号为n的进程的状态段tss，`movw %%dx,%1\n\t"`这条指令就是将当前进程的状态段信息保存到_tmp.b变量中。

tss保存以后，执行了这一句:

```c
xchgl %%ecx,current\n\t"
```

xchgl指令是交换指令，将ecx寄存器的值和current值交换，而ecx寄存器前面讲过对应的是task[n],是要切换过去的进程的task_struct，current是当前进程的task_struct，二者交换，就是将指向正在运行的进程的指针current指向task[n]，这个current指针在进程调度中大有用处。

接下来到了进程切换的最核心的一条指令了:

```c
"ljmp *%0\n\t"
```
ljmp指令是一条跳转指令，它可以从cpu从一行代码跳转到另一行代码执行，跳转分为代码段段内跳转和段间跳转。

段内跳转就是说在同一个代码段中跳转，汇编中可以通过段内跳转实现分支，循环等操作。段间跳转指的是一个进程的代码段跳转到另一个进程的代码段执行。

在汇编语言中，ljmp指令有两种形式:

 - 直接操作数跳转，此时操作数即为目标逻辑地址(选择子，偏移量)， 即形如：ljmp $seg_selector, $offset的方式
 - 使用内存操作数，按照AS手册规定，内存操作数必须使用''做前缀，即形如：ljmp mem48,其中内存位置mem48存放目标逻辑地址：高16bit存放的是set_selector, 低32bit存放的是offset.注意，这条指令的''只是表示间接跳转的意思，与C语言的''完全不同


回到`"ljmp *%0\n\t"`，展开一下就是`"ljmp  *_tmp.a\n\t"`，跳转到_tmp.a的高48位地址，根据struct tmp的定义，_tmp.a就是逻辑地址的偏移地址，_tmp.b就是段选择子，而之前已经执行过`movw %%dx,%1\n\t" \`,_tmp.b的就是进程n的tss段基址，因此ljmp会跳转到进程n的tss段。

ljmp大致过程是：ljmp判断为TSS类型，于是就告诉硬件要切换任务，硬件首先要将当前的PC，esp, eax等现场信息保存在自己的TSS端描述符中，然后再将目标TSS段描述符中的pc, esp, eax的值拷贝至对应寄存器中，这些工作全部做完以后，内核就实现了进程上下文环境的切换。

![在这里插入图片描述](https://img-blog.csdnimg.cn/a905c8f4d79d40348ff7983930d0502d.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAY29kZSBzaG93ZXI=,size_20,color_FFFFFF,t_70,g_se,x_16)


执行完ljmp以后，cpu就跳转到新的进程的代码段执行了，当前switch_to的代码将暂停执行直到进程恢复。


## 4. 总结
总结一下进程切换的过程
1. 比较要切换的进程和当前进程是不是一致，如果一致就不做切换
2. 将要切换的进程的tss段放进_tmp.b变量中
3. 将要切换的进程和当前进程的task_struct交换
4. 执行ljmp指令跳转，ljmp指令判断跳转的地址是tss段，就通知硬件保存当前进程的esp，eax等寄存器值保存到自己的tss段，然后将要切换过去的进程的tss段加载到相应的cpu寄存器完成上下文切换，最后cpu从当前进程代码段切换到新的进程的代码段执行，进程切换完成。
