# Linux
*From Kernel Source Code*

进程与线程(Process and Thread)
----------------------------

进程的内核数据结构：
* 在Linux中，OS为每个进程维护一个数据结构叫做进程描述符，struct task_struct<linux/sched.h>。包含下面这些信息：父子进程指针，虚拟内存指针，信号处理指针，任务状态指针struct tss(后来将该数据结构分配给每个CPU)等。其进程创建过程为fork()-->exec()，由于采用写时拷贝，创建开销小。
* sys_fork、sys_clone、sys_vfork都属于开启新控制流的系统调用，调用栈中都采用do_fork()实现。进程与线程的区别在于do_fork()时传递的参数，故线程又叫LWP(Light Weight Process)。而进程与线程在资源上的开销以及在并发处理时需要注意异步安全的地方就在于这些共享数据。

do_fork函数调用详解：
* copy_process()	/* 调用别的copy函数!! */
* copy_files()		/* 文件描述符fd */
* copy_flags()		/* 调度优先级?? */
* copy_fs()			/* 文件系统挂载信息 */
* copy_mm()			/* 内存描述符 */
* copy_sighand()	/* 信号处理函数 */
* copy_signal()		/* struct signal_struct，包含调度信息等 */

异步安全：
* 涉及到信号处理的IPC(例如socket的IO属于外部中断)需要注意异步信号安全。异步信号有异常与中断两种引发方式，在信号对应的处理函数中要注意全局变量的引用会不会造成死锁或者破坏一致性，而且也要注意抢占式内核可能中断信号处理函数，其它进程也可能引用同一全局数据结构。
* 线程之间共享数据，需要注意线程安全。部分函数会引用同一内核缓冲区，由于该缓冲区用户无法操作故多线程中调用只能加锁，这可能会导致性能问题。

可重入：
* 可重入函数。在信号处理函数中，由于无法预知该函数会在什么时候被调用(例如malloc时被中断)，所以需要考虑函数的可重入性。一般来说引用静态变量、内存操作(malloc/free)、标准IO都是不可重入的。
* 可重入锁。根据tid判断当前请求资源的线程与持有资源的线程是否是同一个，如果是则计数加一无需等待。主要是考虑到互斥锁资源在递归或函数间相互调用时可能发生不必要的死锁。

linux中多任务的实现基于调度器，一种可能的实现方式为借助于时钟中断即硬件中断，周期性地运行调度器以切换控制流。

虚拟内存(Virtual Memory)
-----------------------

cpu总线：地址总线(单向)、数据总线、控制总线
* 8086cpu的地址总线是20位的，ALU是16位的，引入四个段寄存器实现内存寻址
* 80386cpu(i386)是IA32体系结构，内置MMU(Memory Management Unit)以方便内存寻址，为了兼容80286与8086引入了保护模式与实模式。

OS如何将虚拟地址转化为物理地址？
* 逻辑地址 --> 虚拟地址(线性地址) --> 物理地址
* 逻辑地址 --> 虚拟地址：base + offset --> GDT
* 虚拟地址 --> 物理地址：virtual --> PGD + PUD + PMD + PTE + offset
* 线性区描述符：struct vm_area_struct。
* 内存描述符：struct mm_struct。将段虚拟地址统一记录。内存描述符中包含的pgd与segment域为页全局目录与局部描述符表，其中后者在v2.6中被移除。为了查找性能，引入链表与红黑树两种数据结构，并引入缓存机制记录上一个找到的线性区。还有信号量与读写锁保护各种数据结构。线性区描述符内部记录线性区的起始地址以及文件、共享内存等信息，通过struct vm_operations_struct结构体提供一个统一的接口方便内存映射、取消映射、缺页、交换页等处理，非常典型的OOP。
* 为了进一步提高效率，使用TLB(translation lookaside buffer)、多级页表。
* 采用struct page数据结构实现页，unsigned long flags为页的状态，void *virtual为虚拟地址。

内存管理算法：
* 伙伴系统：将连续的不同大小的页串成链表方便分配
* slab分配器：将内存分为固定的大小提高分配速度

虚拟内存的好处：
* 为各种文件设备提供统一的接口
* 方便权限控制
* 方便ipc-shm的实现
* 虚拟地址的分段寻址机制为多级页表的实现提供了基础
* 使得缺页、交换分区等实现对用户透明

如何避免死锁(Dead Lock)
---------------------

锁的作用：维护共享资源的一致性，防止竞态条件。从内核层面看，在实现SMP或内核抢占时需要；从应用层看，使用多线程或者多进程时需要。

锁的原理：所有的锁都需要维护一个可以原子操作的计数变量，在支持SMP的操作系统上，需要提供硬件支持。Linux为此定义了一个atomic_t数据类型，并且在操作时添加了一个LOCK_PREFIX宏，这个宏会根据CONFIG_SMP来扩展。在单核CPU上其是空宏，但是x86-64的SMP上其扩展的内联汇编指令中有一条lock指令，为CPU硬件支持的原子操作，会禁止其它的核去操作该内存地址。早期的Linux由于没有SMP的概念，而且CPU也没有提供硬件支持，故其原子操作只需要注意外部中断的干扰。其一般采用xchg指令实现寄存器到内存的操作，也可以采用cli->sti禁用外部中断，但是后者会造成不必要的麻烦。

锁的分类：根据等待资源时进程的动作可分为semaphore、spin_lock。信号量底层为一个等待队列，获取资源失败的进程会被修改运行状态并且加入等待队列，等资源被归还时会被再次修改运行状态并唤醒；自旋锁底层为"rep;nop"内联汇编，会一直运行。mutex、rw_Lock、RCU也是基于上述两种锁实现的。

死锁产生的必要条件：锁的申请与锁定发生了交叉，即形成了闭环。

建议如下：
* 如果有多个需要保护的数据结构，锁的申请顺序要一致
* 增大锁的使用粒度，将多次锁定的代码封装为新的API，但是要综合权衡性能的下降
* 可以使用try_lock，如果申请不成功，则释放已有的资源
* 不可以递归lock，可能导致递归函数死锁
* 某个模块定义了全局的锁时，注意模块内有锁定的函数不要相互调用
* semaphore与spin_lock不可以同时用。在持有自旋锁时，不要操作内存(kmalloc)，因为底层可能有信号量，在非抢占式内核上可能死锁。因为如果当前持有自旋锁的进程被信号量加入等待队列，而CPU上调度的下一个进程是等待自旋锁的进程，那么就会造成死锁。还有在IO操作等耗时较长的操作中谨慎使用spin_lock锁，会对性能造成不良影响。

关于源代码linux-v0.01
--------------------

共有10239行代码

针对i386CPU编写

关于源代码linux-v1.0.0
---------------------

共有176341行代码，其中kernel/下有6270行代码，include/下有14756行代码

asmlinkage在linux-1.0.0中为空宏

\_\_asm__ \_\_volatile__ 为不可优化的内联汇编代码，有时也采用goto

系统调用通过<kernel/sched.c>中的sys_call_table实现，其为(typedef int (*fn_ptr)())的函数指针,系统调用共有135个

<linux/unistd.h>中定义了 \_\_NR_*syscall*为整型常量，且定义了_syscall*n*(type, name, ...)，为用户空间中直接进行系统调用提供了接口

变参宏的参数提取va_list等是编译器内置函数实现的

原子操作：mutex_atomic_swap，内部使用xchgl对两个值进行交换

调度器内部通过switch_to(tsk)宏定义来切换进程，其扩展为内联汇编代码，最终会跳转到参数的tsk->tss.tr位置执行代码，可以认为sechdule()返回时即为其控制流再次被调度时。这一点对于对于信号量的实现具有重要意义，因为信号量函数在将当前控制流运行状态修改之后加入等待队列，之后运行调度器，再将其移出等待队列。

关于源代码linux-v2.2.0
---------------------

asmlinkage定义为空宏

void schedule(void)进程调度器源码解读：
* 检查current->stat: 如果为TASK_INTERRUPTABLE,并且sig_pending()检查发现有信号，那么设为TASK_RUNNING;如果为TASK_RUNNING，不改变;如果为其它，则del_from_runqueue()。这里可以看到，对于TASK_UNINTERRUPTABLE与其它状态是一样的，说明其是不可打断的睡眠，等待要硬件中断唤醒
* init_task.next_run开始检查其调度信息与调度权重，找到下一个调度进程
* 进程队列由自旋锁runqueue_lock保护，防止调度时新增进程
* struct task_struct *current在arch/asm-i386/current.h中定义为get_current(), 通过%esp寄存器取指针。由于其内核栈按照8KB(IA32上long、int都是4Bytes)对齐(union task_union)，故将%esp(即栈指针地址)低13位清0(即 %esp & (~8091UL[UL是32位??]))。由于在内核栈中，故%esp指向的是内核栈的栈最顶部。该设计针对SMP(Symmetrical Multi-Processing)也是有意义的，因为每个CPU核都有自己的寄存器(不清楚Hyper-Threading)。

关于源代码linux-v2.6.24
----------------------

共有8858692行代码

epoll(eventpoll)
* 底层是callback + poll(IO Multiplexing)
* 回调函数为ep_poll_callback

内核栈采用union thread_union硬编码，起始处为struct thread_info。进程通过struct task_struct中void *stack来寻址内核栈。

汇编语言
-------

Linux内联汇编特点:
* 大部分采用AT&T风格
* AT&T汇编格式: instruction dst, src
* AT&T汇编后缀: b, w, l, q, absq := 字节, 字, 双字, 四字, 绝对四字
* %%为转义的%

汇编指令解析: 

	__asm__ __volatile__(
		assembly template : 
		output operand list : 
		input operand list : 
		clobber list
	);

	cli: clear (external) interruption
	sti: set (external) interruption
	clts: clear task
	je: jump if equal
	jne: jump if not equal
	lidt: load IDT to IDTR
	lgdt: load GDT to GDTR
	out: write to I/O port
	in: read from I/O port
	jns: jump if not negative

名词解释
-------

RCU: Read-Copy Update

GDT: Global Descriptor Table

LDT: Local Descriptor Table

IDT: Interrupt Descriptor Table

PGD: Page Global Directory

PUD: Page Upper Directory

PMD: Page Middle Directory

PTE: Page Table Entry

PTER: Page Table Entry Register

IDTR: Interrupt Descriptor Register

GDTR: Global Descriptor Register

LRU: Least Recently Used