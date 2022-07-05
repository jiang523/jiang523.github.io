---
layout: post
title: 进程的创建
subtitle: fork函数
categories: Linux
tags: [Linux]
---

 ## 1. 进程的信息
 ### 1.1 进程的结构


 在Linux中，一切皆文件，进程也是保存在内存中的一个实例，下图描述了进程的结构: ![image-20220624110006061](https://jiang523.github.io//images/2022-06-23-linux-fork/image-20220624110006061.png)
 - 堆栈:保存局部变量
 - 数据段:一般存放全局变量和静态变量
 - 代码段:存储进程的代码文件
 - TSS状态段:进程做切换时，需要保存进程现场以便恢复，一般存储一些寄存器的值。
 - task_struct : 进程的结构体描述符，用来描述一个进程的属性，这是一种面向对象的编程思想，task_struct的结构大致如下:

```c
struct task_struct {
	
	long state;	//* -1 unrunnable, 0 runnable, >0 stopped * 进程的状态
	long counter; // 时间片
	long priority; //优先级
	long signal; //信号
	struct sigaction sigaction[32];
	long blocked;
	int exit_code;
	unsigned long start_code,end_code,end_data,brk,start_stack;
	long pid,father,pgrp,session,leader; //进程号，父进程号，会话等
	unsigned short uid,euid,suid;
	unsigned short gid,egid,sgid;
	long alarm;  //警告
	long utime,stime,cutime,cstime,start_time;//用户态、内核态执行时间
	unsigned short used_math;
	int tty;
	unsigned short umask;
	struct m_inode * pwd;
	struct m_inode * root;
	struct m_inode * executable;
	unsigned long close_on_exec;
	struct file * filp[NR_OPEN]; //打开的文件列表
	struct desc_struct ldt[3]; //ldt段
	struct tss_struct tss; //tss段
};

```

通过管理进程对应的task_struct，可以完成进程的相关操作。

 ### 1.2 GDT和LDT

操作系统在保护模式下，内存管理分为分段模式和分页模式。分段模式下内存的寻址为「段基址:偏移地址」。对一个段的描述包括以下三个方面：【Base Address,Limit,Access】，他们加在一起被放在一个64bit长的数据结构中，被称为段描述符。因此需要用64bit的寄存器去存储段描述符，但是操作系统的段基址寄存器只能存储16bit的数据，因此无法直接存储64bit的段描述符，为了解决这个问题，操作系统将段描述符存放在一个全局的数组中，而段寄存器直接存储对应的段描述符对应的下标，这个全局的数组叫做GDT

   由于GDT也需要直接存在内存中，所以操作系统用GDTR寄存器来存储GDT的基地址，因此寻  址的过程为:

	1. 通过GDTR寄存器找到GDT的基地址
	2. 通过段寄存器找到段描述符的索引
	3. 通过GDT基地址+索引从GDT数组中找到段描述符
	4. 通过段描述符基地址+偏移地址找到线性地址

至于LDT,本质上和GDT是类似的，但也有不一样的地方，LDT本身也是一段内存，因此需要段描述符去描述它，它的段描述符存在GDT中，而LDT有LDTR寄存器，LDTR并不存储LDT的段基址，而是一个段选择子，是LDT的索引。

   用一张图来诠释GDT和LDT的寻址过程:
   ![GDT和LDT](https://img-blog.csdnimg.cn/58db57f78f964a1ead83dc27601ebfb3.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAcXFfNDE3NjExNzY=,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

 ### 1.3 进程的状态


 进程一共有五种状态，分别是:
  1. TASK_RUNNING 运行状态

     运行状态表示正在运行，只有这个状态的进程才能被执行

  2. TASK_INTERRUPTIBLE 可中断睡眠状态

     当一个进程处于可中断睡眠状态时，它不会占用cpu资源，但是它可以响应中断或者信号。如socket等待连接建立时，它是睡眠的，但是连接一旦建立就会被唤醒。这种状态就是阻塞状态。

  3. TASK_UNINTERRUPTIBLE  不可中断睡眠状态

     和TASK_INTERRUPTIBLE不同，处于TASK_UNINTERRUPTIBLE状态的进程无法被中断或者信号唤醒，假设一个进程是TASK_UNINTERRUPTIBLE状态，你会惊奇的发现通过kill -9无法杀死该进        程，因为它无法响应异步信号。这种状态很少见，一般发生在内核态程序中，如读取某个设备的文件，需要通过read系统调用通过驱动操作硬件设备读取，这个过程是无法被中断的。

  4. TASK_ZOMBIE  僵死状态

     处于TASK_ZOMBIE状态的进程并不代表着进程已经被销毁，此时除了task_struct，进程占有的所有资源将被释放，之所以不释放task_struct是因为task_struct保存着一些统计信息，其父进程      可能需要这些信息。

  5. TASK_STOPPED

     不保留task_struct，进程资源全部被释放
 ## 2. 系统初始化——main函数
 上面大致介绍了和进程相关的一些信息说明，本节将从kernel的main.c方法开始，分析进程的创建过程.

Linux的main.c文件，是Linux开机时内核初始化函数，在初始化的过程中，内核将创建系统的第一个进程:0号进程，0号进程不做任何操作，也不能被终止(除非系统异常或者关机),以后创建的每一个进程都是0号进程的子孙进程。

main.c的入口是main()函数，main()函数的主要实现:

```c
void main(void){
 	ROOT_DEV = ORIG_ROOT_DEV;
 	drive_info = DRIVE_INFO; 
 	
    //省略了一段内存初始化操作
  
	mem_init(main_memory_start,memory_end);  // 主内存区初始化
	trap_init();                             // 陷阱门(硬件中断向量)初始化
	blk_dev_init();                          // 块设备初始化
	chr_dev_init();                          // 字符设备初始化
	tty_init();                              // tty初始化
	time_init();                             // 设置开机启动时间 startup_time
	sched_init();                            // 调度程序初始化(加载任务0的tr,ldtr)
	buffer_init(buffer_memory_end);          // 缓冲管理初始化，建内存链表等。
	hd_init();                               // 硬盘初始化
	floppy_init();                           // 软驱初始化
	sti();                                   // 所有初始化工作都做完了，开启中断
    // 下面过程通过在堆栈中设置的参数，利用中断返回指令启动任务0执行。
	move_to_user_mode();                     // 移到用户模式下执行
	if (!fork()) {		
		init();                             // 在新建的子进程(任务1)中执行。
	}
	for(;;) pause();
}
```
我们来看一看和进程管理相关的两个初始化过程:
1. time_init()
```c
static void time_init(void)
{
	struct tm time;
	do {
		time.tm_sec = CMOS_READ(0);
		time.tm_min = CMOS_READ(2);
		time.tm_hour = CMOS_READ(4);
		time.tm_mday = CMOS_READ(7);
		time.tm_mon = CMOS_READ(8);
		time.tm_year = CMOS_READ(9);
	} while (time.tm_sec != CMOS_READ(0));
	BCD_TO_BIN(time.tm_sec);
	BCD_TO_BIN(time.tm_min);
	BCD_TO_BIN(time.tm_hour);
	BCD_TO_BIN(time.tm_mday);
	BCD_TO_BIN(time.tm_mon);
	BCD_TO_BIN(time.tm_year);
	time.tm_mon--;                              // tm_mon中月份的范围是0-11
	startup_time = kernel_mktime(&time);        // 计算开机时间
}
```

 time_init()函数主要从CMOS管中读取一个实时时钟的年月日时分秒等信息并保存起来，并且通过kernel_mktime()函数来计算一个startup_time作为系统的开机时间,而kernel_time()函数就是根据从CMOS读出的信息计算出1970年1月1日到现在的一个时间。

分析这个函数主要想介绍一下内核一个很重要的时间概念:jiffies(系统滴答)
jiffies是系统的脉搏，或者说是系统的节拍。在内核中，系统会以一定的频率发生定时中断，也就是说，某个进程正在运行，运行一段时间后系统会暂停这个进程运行，然后切换到另一个进程运行，jiffies决定了系统发生中断的频率，因此它和进程的调度息息相关。

2. sched_init()

```c
void sched_init(void)
{
	int i;
	struct desc_struct * p;                 // 描述符表结构指针
	
	if (sizeof(struct sigaction) != 16)         // sigaction 是存放有关信号状态的结构
		panic("Struct sigaction MUST be 16 bytes");

	set_tss_desc(gdt+FIRST_TSS_ENTRY,&(init_task.task.tss));
	set_ldt_desc(gdt+FIRST_LDT_ENTRY,&(init_task.task.ldt));

	p = gdt+2+FIRST_TSS_ENTRY;
	for(i=1;i<NR_TASKS;i++) {
		task[i] = NULL;
		p->a=p->b=0;
		p++;
		p->a=p->b=0;
		p++;
	}
}
```
首先来看一个变量task[]，它的定义为:

```c
struct task_struct * task[NR_TASKS] = {&(init_task.task), }
```
在sched.c文件中，定义了一个task数组，这个数组的类型为task_struct结构体，它的最大容量NR_TASKS=64,它的初始值也就是task[0] = init_task.task。

这个task数组的意义是:

> task是一个保存进程结构体的数组，最大容量为64，task[0]的位置保存了0号进程task_struct

再回到sched_init()的代码，首先定义了一个desc_struct指针，然后为0号进程设置了它的ldt段和tss段，ldt段是由数据段和代码段构成的。然后从1开始遍历task数组，将每个槽设置为null，并将其gdt设置为空，由于是从1开始遍历，因此处于index=0的0号进程不会被置空。可见，sched_init()函数创建了0号进程。

## 3.fork()函数

进行一系列初始化后，执行了这句代码:
```c
move_to_user_mode();    
```
这个函数是将当前模式由内核态转为用户态。

> 内核态:不可抢占的          
> 用户态:可抢占，可以进行调度的

也就是说，上述所有的初始化操作都是在内核态执行的，这么做的目的是，内核初始化过程是不能被中断的，在内核态运行可以保证这一点。

切换到用户态以后，便开始创建进程了:

```c
if (!fork()) {		
	init();                           
}
```
这里的fork()函数，就是linux创建进程的函数，进入这个函数，它的声明为:

```c
static inline _syscall0(int,fork)
```

> syscall是系统调用函数，就是内核自己实现的一些函数，如 read，open,chmod等，这些函数可以直接提供给开发人员调用，而调用的过程需要切换到内核态进行，因为函数调用过程中不允许被中断。这个调用过程称为系统调用。

_syscall0的函数定义如下:

```c
#define _syscall0(type,name) \
type name(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80"\
	: "=a" (__res)              \
	: "0" (__NR_##name));       \
if (__res >= 0) \
	return (type) __res; \
errno = -__res; \
return -1; \
}
```

把参数替换成fork以后，是这个样子:


```c
#define _syscall0(type,fork) \
type name(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80"\
	: "=a" (__res)              \
	: "0" (__NR_FORK));       \
if (__res >= 0) \
	return (type) __res; \
errno = -__res; \
return -1; \
}
```
这是一段汇编代码，这段代码的执行过程是这样的:
1. 将_res变量和eax寄存器绑定，后面_res变量的值就是从eax寄存器读出来的值
2. 将_NR_FORK=2 赋值给eax寄存器
3. "int $0x80"产生一个软中断，由于之前sched_init()函数中设置了0x80中断服务函数:
set_system_gate(0x80,&system_call),因此产生0x80中断以后会调用system_call，system_call在system_call.s中定义，

```c
system_call:
	cmpl $nr_system_calls-1,%eax    # 调用号如果超出范围的话就在eax中置-1并退出
	ja bad_sys_call
	push %ds                        # 保存原段寄存器值
	push %es
	push %fs
	pushl %edx
	pushl %ecx		
	pushl %ebx		
	movl $0x10,%edx	
	mov %dx,%ds
	mov %dx,%es
	movl $0x17,%edx
	mov %dx,%fs
	call sys_call_table(,%eax,4)        # 间接调用指定功能C函数
	pushl %eax                          # 把系统调用返回值入栈
	movl current,%eax                   # 取当前任务(进程)数据结构地址→eax
	cmpl $0,state(%eax)	
	jne reschedule
	cmpl $0,counter(%eax)
	je reschedule

ret_from_sys_call:
	movl current,%eax		# task[0] cannot have signals
	cmpl task,%eax
	je 3f                   # 向前(forward)跳转到标号3处退出中断处理
	cmpw $0x0f,CS(%esp)	
	jne 3f
	cmpw $0x17,OLDSS(%esp)	
	jne 3f
	movl signal(%eax),%ebx          # 取信号位图→ebx,每1位代表1种信号，共32个信号
	movl blocked(%eax),%ecx         # 取阻塞(屏蔽)信号位图→ecx
	notl %ecx                       # 每位取反
	andl %ebx,%ecx                  # 获得许可信号位图
	bsfl %ecx,%ecx                  # 从低位(位0)开始扫描位图，看是否有1的位，若有，则ecx保留该位的偏移值
	je 3f                           # 如果没有信号则向前跳转退出
	btrl %ecx,%ebx                  # 复位该信号(ebx含有原signal位图)
	movl %ebx,signal(%eax)          # 重新保存signal位图信息→current->signal.
	incl %ecx                       # 将信号调整为从1开始的数(1-32)
	pushl %ecx                      # 信号值入栈作为调用do_signal的参数之一
	call do_signal                  # 调用C函数信号处理程序(kernel/signal.c)
	popl %eax                       # 弹出入栈的信号值
3:	popl %eax                       # eax中含有上面入栈系统调用的返回值
	popl %ebx
	popl %ecx
	popl %edx
	pop %fs
	pop %es
	pop %ds
	iret
```

首先将各寄存器入栈，然后调用了关键的一个函数:

```c
call sys_call_table(,%eax,4)
```
sys_call_table是一个数组，在sys.h中定义,它保存着所有系统调用的函数名，%eax就是eax寄存器的值，前面提到过0x80中断产生之前将_NR_FORK=2加入到了eax寄存器中，因此调用的就是sys_call_table[2]，也就是sys_fork函数:
```c
sys_fork:
	call find_empty_process
	testl %eax,%eax            
	js 1f
	push %gs
	pushl %esi
	pushl %edi
	pushl %ebp
	pushl %eax
	call copy_process
	addl $20,%esp   
1:	ret
```
这是一段汇编代码，首先调用了fork.c文件中的find_empty_process函数，目的在于从task进程数组中找到一个空的槽用于保存要创建的进程的task_struct，这个函数会返回进程的pid。


testl %eax,%eax 指令作用是将call find_empty_process函数的返回值保存到eax寄存器中。随后进行了一系列寄存器数的压栈，最后调用了copy_process()函数:

```c
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
		long ebx,long ecx,long edx,
		long fs,long es,long ds,
		long eip,long cs,long eflags,long esp,long ss)
{
	struct task_struct *p;
	int i;
	struct file *f;
	p = (struct task_struct *) get_free_page();
	if (!p)
		return -EAGAIN;
	task[nr] = p;
	*p = *current;	
	p->state = TASK_UNINTERRUPTIBLE;
	p->pid = last_pid;              // 新进程号。也由find_empty_process()得到。
	p->father = current->pid;       // 设置父进程
	p->counter = p->priority;       // 运行时间片值
	p->signal = 0;                  // 信号位图置0
	p->alarm = 0;                   // 报警定时值(滴答数)
	p->leader = 0;		
	p->utime = p->stime = 0;        // 用户态时间和和心态运行时间
	p->cutime = p->cstime = 0;      // 子进程用户态和和心态运行时间
	p->start_time = jiffies;        // 进程开始运行时间(当前时间滴答数)
	p->tss.back_link = 0;
	p->tss.esp0 = PAGE_SIZE + (long) p;     // 任务内核态栈指针。
	p->tss.ss0 = 0x10;                      // 内核态栈的段选择符(与内核数据段相同)
	p->tss.eip = eip;            // 指令代码指针
	p->tss.eflags = eflags;     // 标志寄存器
	p->tss.eax = 0;            // 这是当fork()返回时新进程会返回0的原因所在
	p->tss.es = es & 0xffff;      // 段寄存器仅16位有效

	p->tss.ldt = _LDT(nr);  // 任务局部表描述符的选择符(LDT描述符在GDT中)
	p->tss.trace_bitmap = 0x80000000;   // 高16位有效
	if (copy_mem(nr,p)) {
		task[nr] = NULL;
		free_page((long) p);
		return -EAGAIN;
	}
	set_tss_desc(gdt+(nr<<1)+FIRST_TSS_ENTRY,&(p->tss));
	set_ldt_desc(gdt+(nr<<1)+FIRST_LDT_ENTRY,&(p->ldt));
	p->state = TASK_RUNNING;	/* do this last, just in case */
	return last_pid;
}
```

这个函数非常长，省略了一些无关代码，主要有以下几个重要步骤:
1. 定义了一个task_struct *p，并为其分配了内存，然后放到task数组对应的槽当中
2. 将当前进程指针赋值给p

```c
  *p = *current
```
*current指向当前进程，也就是调用fork函数的进程，本过程中就是0号进程，*p指向要创建的进程，本过程中就是1号进程。*p=*current，这不就是将0号进程的task_struct直接赋值给了1号进程吗？原来进程的创建第一步都是先把它的父进程拿来拷贝一份。

3. 将进程p的状态设置为不可中断睡眠状态

```c
p->state = TASK_UNINTERRUPTIBLE
```
这么做的目的是当前进程既不能处理信号，也无法参与调度。

4. 设置task_struct特定属性
要创建的进程p是通过拷贝父进程task_struct而来，但是作为一个进程，它需要有自己特定的属性，因此需要对其特定的属性进行赋值:

```c
    p->pid = last_pid;              // 新进程号。也由find_empty_process()得到。
	p->father = current->pid;       // 设置父进程
	p->counter = p->priority;       // 运行时间片值
	p->signal = 0;                  // 信号位图置0
	p->alarm = 0;                   // 报警定时值(滴答数)
	p->utime = p->stime = 0;        // 用户态时间和和内核运行时间
	p->cutime = p->cstime = 0;      // 子进程用户态和和内核运行时间
	p->start_time = jiffies;        // 进程开始运行时间(当前时间滴答数)
	p->tss.esp0 = PAGE_SIZE + (long) p;     // 任务内核态栈指针。
	p->tss.ss0 = 0x10;                      // 内核态栈的段选择符(与内核数据段相同)
	p->tss.eip = eip;            // 指令代码指针
	p->tss.eflags = eflags;     // 标志寄存器
	p->tss.eax = 0;           //eax寄存器
	p->tss.ldt = _LDT(nr);  // 任务局部表描述符的选择符(LDT描述符在GDT中)
```
大部分属性都容易看懂，但是有两个地方却暗藏玄机:

```c
p->tss.eax = 0;
p->tss.eip = eip;
```
看似很常规的两行代码:  将子进程的eax寄存器设置为0，将子进程eip寄存器设置为父进程的eip寄存器值。

eax寄存器存储着函数的返回值，而eip寄存器，存储着cpu要去读取的下一行指令代码的位置，那寻根溯源一下，父进程下一行代码是哪一行呢?

事实上，我们正在分析的fork系统调用的代码并不属于父进程执行的代码，它属于内核态程序，真正父进程执行的代码应该是它产生软中断而调用fork系统调用的下一行，也就是这个:

```c
#define _syscall0(type,fork) \
type name(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80"\
	: "=a" (__res)              \
	: "0" (__NR_FORK));       \
if (__res >= 0) \
	return (type) __res; \
errno = -__res; \
return -1; \
```

中的这一行

```c
if (__res >= 0)
```
这也就意味着，当子进程开始运行的时候，会从这一行开始执行。

5. 设置进程的tss和ldt

```c
set_tss_desc(gdt+(nr<<1)+FIRST_TSS_ENTRY,&(p->tss));
set_ldt_desc(gdt+(nr<<1)+FIRST_LDT_ENTRY,&(p->ldt));
```

6. 将进程p状态改为运行状态

```c
p->state = TASK_RUNNING;
```
进程创建到这里，已经为要创建的进程创建了task_struct，并完成了task_struct初始化，也设置了进程的ldt段和tss段，那么这个进程已经可以开始运行并可以参与调度了，因此将进程设置为就绪状态。

7. 返回进程id
```c
return last_pid
```
就是返回子进程的id，这里返回的是一号进程的id 1。

copy_process()函数返回了，sys_fork也就返回了，返回的值就是copy_process函数的返回值，这里就是一号进程的id=1,然后就返回到system_call执行，将各寄存器值出栈，然后0x80中断就返回了,将切换到用户态继续执行0号进程的代码，_syscall0将继续往下执行

```c
#define _syscall0(type,fork) \
type name(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80"\
	: "=a" (__res)              \
	: "0" (__NR_FORK));       \
if (__res >= 0) \
	return (type) __res; \
errno = -__res; \
return -1; \
}
```
前面提到过_res的值就是eax,当前0号进程eax值就是sys_fork调用的copy_process()的返回值，前面提到了copy_process()返回的是子进程1号进程的进程id，也就是1。因此if条件符合，_syscall0返回1。

到这里fork()函数执行完毕并返回了1，回到fork调用的地方

```c
if (!fork()) {
	init();                            
}
for(;;) pause();
```

由于fork()返回了1，所以这里if条件不符合,往下走到一段死循环，循环里调用了pause()函数,点开pause函数的声明:

```c
static inline _syscall0(int,pause)
```

看到这里立马就懂了,pause()函数也是一个系统调用,省去中间系统调用的过程，直接来到pause调用的函数:

```c
int sys_pause(void)
{
	current->state = TASK_INTERRUPTIBLE;
	schedule();
	return 0;
}
```
与前面fork系统调用不同的是，pause系统调用是c语言实现的。pause先将进程状态设置为可中断睡眠状态，然后进行了一次schedule()也就是进行了一次进程调度，由于当前只有0号和1号两个进程，所以进程调度的结果肯定是由0号进程切换到1号进程，至于进程的调度和进程的切换，后面会有详细介绍这里就不展开了。

现在正在运行的是1号进程，cpu就会找到gdt找到1号进程的ldt，就会读取1号进程的eip寄存器去读取指令。现在重点来了，我们在介绍0号进程fork1号进程的时候提示过，当时0号进程将自己的eip寄存器值赋给了1号进程，所以1号进程eip寄存器存储的下一行代码是:

```c
if (__res >= 0)
```
_res是eax寄存器的值，而fork的时候0号进程将1号进程的eax寄存器值得设置为了0

```c
p->tss.eax = 0;
```
因此if条件也是符合的，就将_res 返回了，返回到哪里了呢?返回的肯定是fork()函数被调用的地方，也就是:

```c
if (!fork()) {
	init();                            
}
for(;;) pause();
```
看到这你可能有点懵，怎么又到这来了，0号进程不是已经执行过一次了吗，又来。。

但是和之前不同的是，这里的fork()返回的_res值是0，是符合if条件的，然后会执行init()。。

这就是fork()函数很神秘的地方，它实现了一个函数"return了两次"，一次是父进程返回，一次是子进程返回。

至于init()函数，里面涉及了一些shell初始化，tty0初始化，输入输出设备初始化操作，跟进程的创建没有太大的关系，就不继续展开了。

## 4. 总结

本文以0号进程创建1号进程为例，分析了fork()函数详细的过程，有以下:

1. sched_init()定义了task[64]，并将第0个位置保存0号进程task_struct，随后0号进程被创建
2. 从内核态切换到用户态，开始运行0号进程
3. 0号进程调用fork()系统调用，产生一个0x80软中断，调用了sys_fork函数
4. sys_fork先为1号进程分配了id，然后调用了copy_process函数
5. copy_process函数将0号进程的task_struct赋值给1号进程，然后设置了1号进程特定的属性，并设置了eip和eax寄存器
6. 1号进程创建完毕，返回0号进程执行，0号进程调用pause()休眠，进程调度到了1号进程执行
7. 1号进程返回_res=0，然后执行init()
