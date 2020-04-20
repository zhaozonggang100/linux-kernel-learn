### 1、Linux涉及到栈的种类  

`内核栈、中断栈、用户栈 `

- 1、x86  

	1、x86架构中中断有自己中断栈  
	2、用户栈切换到内核栈  
		参考：http://abcdxyzk.github.io/blog/2015/06/02/kernel-sched-user-to-kernel

	thread_info和内核栈共同占用2个连续的物理页框（2*4k）  
```
    union thread_union {
    #ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK
    	struct task_struct task;
    #endif
    #ifndef CONFIG_THREAD_INFO_IN_TASK
    	struct thread_info thread_info;
    #endif
    	unsigned long stack[THREAD_SIZE/sizeof(long)];
    };
```

	esp寄存器是CPU栈指针，存放内核栈顶地址，从用户态切换到内核态时，进程的内核栈总是空的，此时esp指向内核栈栈顶，系统调用公用部分会将用户栈的%esp压入内核栈栈顶，通过iret指令返回用户空间时会弹出用户栈寄存器的值（%esp）和寄存器状态  
	
	x86体系每一个CPU包含一个TSS（任务状态段）结构，TSS段的地址保存在tr寄存器中，tss段包含了当前cpu运行进程的内核栈指针，当从用户态切换到内核态会从tss段中取出当前进程的内核栈，然后把用户栈对应的esp寄存器中的值压进内核栈，再把TSS中进程内核栈的值赋值给ESP寄存器
	
	tss段中espo代表进程内核栈

- 2、ARM  

	arm架构中中断使用被中断进程的内核栈作为中断栈  