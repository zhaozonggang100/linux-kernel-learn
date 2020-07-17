### 1、概述

### 2、rest_init函数原型

```c
// file：init/main.c
static noinline void __ref rest_init(void)
{
    struct task_struct *tsk;
    int pid;

    rcu_scheduler_starting();
    /*
    	创建内核线程kernel_init
    */
    /*
     * We need to spawn init first so that it obtains pid 1, however
     * the init task will end up wanting to create kthreads, which, if
     * we schedule it before we create kthreadd, will OOPS.
     */
    /*
    	idle线程（0号线程）创建Linux中的1号线程kernel_init
    	在用户空间加载init程序之后init程序变为1号进程，init负责创建并监视其他的进程
    */
    pid = kernel_thread(kernel_init, NULL, CLONE_FS);
    /*
     * Pin init on the boot CPU. Task migration is not properly working
     * until sched_init_smp() has been run. It will set the allowed
     * CPUs for init to the non isolated CPUs.
     */
    rcu_read_lock();
    tsk = find_task_by_pid_ns(pid, &init_pid_ns);
    set_cpus_allowed_ptr(tsk, cpumask_of(smp_processor_id()));
    rcu_read_unlock();
    
    numa_default_policy();
    /*
    	https://www.cnblogs.com/embedded-linux/p/6618717.html
    	idle线程（0号线程）创建Linux中的2号线程，会管理所有的内核线程，循环调度内核线程
    	kthread_create创建一个内核线程会唤醒kthreadd内核线程将新的内核线程加入全局列表
    	所有内核线程的ppid都是2
    */
    pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    rcu_read_lock();
    kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
    rcu_read_unlock();
    
    /*
     * Enable might_sleep() and smp_processor_id() checks.
     * They cannot be enabled earlier because with CONFIG_PRREMPT=y
     * kernel_thread() would trigger might_sleep() splats. With
     * CONFIG_PREEMPT_VOLUNTARY=y the init task might have scheduled
     * already, but it's stuck on the kthreadd_done completion.
     */ 
    /*
    	将系统状态设置为可调度，这样就可以执行进程调度了
    */
    system_state = SYSTEM_SCHEDULING;

    complete(&kthreadd_done);

    /*
     * The boot idle thread must execute schedule()
     * at least once to get things moving:
     */
    schedule_preempt_disabled();
    /* Call into cpu_idle with preempt disabled */
    cpu_startup_entry(CPUHP_ONLINE);
}
```

### 3、kernel_init加载内核模块、初始化init进程

```c
static int __ref kernel_init(void *unused)
{   
    int ret;
    
    /*
    	该函数会初始化内核模块
    	kernel_init_freeable-->do_basic_setup-->do_initcalls
    	
    	// https://www.cnblogs.com/sky-heaven/p/10344823.html
    	static void __init do_initcalls(void)
        {
            int level;
			
            for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
                do_initcall_level(level);
        }
        
        // 执行每个initcall段中的所有函数
        static void __init do_initcall_level(int level)
        {
            initcall_t *fn;

            strcpy(initcall_command_line, saved_command_line);
            parse_args(initcall_level_names[level],
                   initcall_command_line, __start___param,
                   __stop___param - __start___param,
                   level, level,
                   NULL, &repair_env_string);

			// initcallrotfs会在initcall5.init~initcall6.init之间执行
            for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
                do_one_initcall(*fn);
        }
        
        // 下面的结构体数组会放在.init.data section中，这是一个指针数组
        static initcall_t *initcall_levels[] __initdata = {
            __initcall0_start,
            __initcall1_start,
            __initcall2_start,
            __initcall3_start,
            __initcall4_start,
            __initcall5_start,
            __initcall6_start,
            __initcall7_start,
            __initcall_end,
        };
        
        // __initcall0_start在vmlinux.lds中定义为：.initcall0.init 段
        extern initcall_t __initcall_start[];
        extern initcall_t __initcall0_start[];
        extern initcall_t __initcall1_start[];
        extern initcall_t __initcall2_start[];
        extern initcall_t __initcall3_start[];
        extern initcall_t __initcall4_start[];
        extern initcall_t __initcall5_start[];
        extern initcall_t __initcall6_start[];
        extern initcall_t __initcall7_start[];
        extern initcall_t __initcall_end[];
    */
    kernel_init_freeable();
    /* need to finish all async __init code before freeing the memory */
    async_synchronize_full();
    ftrace_free_init_mem();
    free_initmem();
    mark_readonly();
    // 将系统设置为运行态
    system_state = SYSTEM_RUNNING;
    numa_default_policy();
    
    rcu_end_inkernel_boot();
    place_marker("M - DRIVER Kernel Boot Done");

    /*
    	https://blog.csdn.net/kevin_hcy/article/details/17663341
    	如果虚拟文件系统（initrd、ramdisk）中存在ramdisk_execute_command中定义的文件，则直接执行对应的初始化文件
    */
    if (ramdisk_execute_command) {
        ret = run_init_process(ramdisk_execute_command);
        if (!ret)
            return 0;
        pr_err("Failed to execute %s (error %d)\n",
               ramdisk_execute_command, ret);
    }

    /*
     * We try each of these until one succeeds.
     * 
     * The Bourne shell can be used instead of init if we are
     * trying to recover a really broken machine.
     */
    if (execute_command) {
        ret = run_init_process(execute_command);
        if (!ret)
            return 0;
        panic("Requested init %s failed (error %d).",
              execute_command, ret);
    }
    if (!try_to_run_init_process("/sbin/init") ||
        !try_to_run_init_process("/etc/init") ||
        !try_to_run_init_process("/bin/init") ||
        !try_to_run_init_process("/bin/sh"))
        return 0;

    panic("No working init found.  Try passing init= option to kernel. "
          "See Linux Documentation/admin-guide/init.rst for guidance.");
}
```

