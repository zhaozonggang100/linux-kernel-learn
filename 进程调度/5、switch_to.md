### 1、arm64 上下文切换  
https://blog.csdn.net/gaojy19881225/article/details/84999971

https://www.cnblogs.com/linganxiong/p/9119686.html（arm64汇编）

```
调度点：
	1、kernel返回user检查是否可抢占current，设置抢占标志，在时钟滴答中检查抢占标志执行schedule
	2、硬件中断返回内核检查是否可抢占current，设置抢占标志，在时钟滴答中检查抢占标志执行schedule
	3、软件中断返回内核检查是否可抢占current，设置抢占标志，在时钟滴答中检查抢占标志执行schedule
	4、用户主动调休眠调用schedule
要点：
	1、内核线程没有进程地址空间即task_struct->mm为NULL，但是内核线程被调度时需要进程地址空间，这时内核线程的task_struct->active_mm是指向被切换的进程的地址空间(task_struct->mm)，用户进程的mm和active_mm都指向用户地址空间
	2、所有用户进程拥有相同的内核地址空间，在内核态进行上下文切换时，内核线程的active_mm切换为被切换进程的地址空间的active_mm，此时进程上下文还没切换完成，但是由于在内核态，内核地址空间是相同的，所以不影响接下来的其他上下文切换
	3、切换时候要先获取当前调用schedule的cpu，然后获取该cpu的rq（运行队列）
	4、如果被切换的是内核线程，切入的也是内核线程，active_mm需要在rq上找一次使用的mm_struct
	5、pgd地址，用户pgd在cpu核的ttbr0_el1，内核pgd在cpu核的ttbr1_el1

函数执行：
schedule-->__schedule-->context_switch-->switch_to-->__switch_to
1、__switch_to内部cpu_switch_to进行上下文（寄存器）保存，最后执行ret指令，在_do_fork
->copy_process->copy_thread_tls->copy_thread
2、在copy_thread中将ret_from_fork赋值给tsk->thread->pc，然后进入cpu_switch_to
3、当cpu_switch_to调用ret指令时将PC指针赋值给lr，将lr赋值给pc
4、子进程返回之后执行的就是ret_from_fork，父进程回到调用_do_fork的地址
	对于_do_fork创建内核线程，返回到[create_kthread-->kernel_thread->]kthread
	对于user进程返回到用户空间调用fork函数的地方，会返回两次
	
切换内容：
1、地址空间寄存器，切换进程地址空间（tsk->mm、tsk->active_mm），实际上是切换ttbr0_el1
2、通用寄存器（Xx）
3、浮点寄存器
4、其他寄存器（ASID、thread process ID register等）
```

### 2、X86_64上下文切换

