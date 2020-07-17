### 1、kernel_start

```c
// file:init/main.c
asmlinkage __visible void __init start_kernel(void)
{
    char *command_line;
    char *after_dashes;

    // idle进程栈写入魔术：STACK_END_MAGIC(0x57AC6E9D)
    set_task_stack_end_magic(&init_task);
    /*
    	获取当前运行的cpu，然后会打印Booting Linux on physical CPU 0xX
    */
    smp_setup_processor_id();
    /*
    	初始化全局的hash数组
    	struct debug_bucket {
            struct hlist_head   list;
            raw_spinlock_t      lock;
        };
        static struct debug_bucket  obj_hash[ODEBUG_HASH_SIZE];
    */
    debug_objects_early_init();

    /*
    	cgroup初始化，要比其他子系统早
    */
    cgroup_init_early();

    /*
    	关闭本地中断，arm64汇编指令
    	arm32是cpsr寄存器保存了cpu的状态，可以用msr、mrs读写cprs寄存器，比特位7代表中断标志位
    	arm64是
    	static inline void arch_local_irq_disable(void)
        {
            asm volatile(
            	https://blog.csdn.net/longwang155069/article/details/52151595
            	设置DAIF中的I位为1，也就是第二位，代表中断标志位
                "msr    daifset, #2     // arch_local_irq_disable"
                :
                :
                : "memory");
        }
        https://www.cnblogs.com/elnino/p/4313340.html（asm内嵌汇编）
        __asm__ ("InstructionList"						// 指令
                                    :Output				// 输出
                                    :Input				// 输入
                                    :Clobber/Modify);	// 操作的存储设备
    */
    local_irq_disable();
    early_boot_irqs_disabled = true;

    /*   
     * Interrupts are still disabled. Do necessary setups, then
     * enable them.
     * 如果上面不关闭中断，初始化cpu时，中断处理就可能出现意想不到的结果
     */
    // 初始化cpu，激活boot cpu
    boot_cpu_init();
    // arm64没有highmem，该函数是空操作
    page_address_init();
    // 打印Linux开机信息
    pr_notice("%s", linux_banner);
    /*
    	比较重要的函数，会根据command_line初始化arch相关的结构
    	内存：memblock、fixmap、ioremap等
    	https://blog.csdn.net/GerryLee93/article/details/106476252?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.nonecase
    */
    setup_arch(&command_line);
    /*   
     * Set up the the initial canary and entropy after arch
     * and after adding latent and command line entropy.
     */
    add_latent_entropy();
    add_device_randomness(command_line, strlen(command_line));
    /*
    	参考：https://blog.csdn.net/yin262/article/details/46714181
    	初始化“金雀丝”，防止栈溢出
    */
    boot_init_stack_canary();
    /*
    	参考：https://blog.csdn.net/u012171620/article/details/73277029
    	清除mm_struct中的cpu位图，说明该mm_struct与cpu没关系
    	具体不太清除干了啥，可能是进程的内存不缓存cpu？？？
    */
    mm_init_cpumask(&init_mm);
    setup_command_line(command_line);
    // smp架构设置各个cpu的id，单核的是空操作
    setup_nr_cpu_ids();
    /*
    	https://blog.csdn.net/yin262/article/details/46787879
    	smp中有效，为内存初始化准备，创建per-cpu变量
    	所有cpu的变量是per-cpu数组中的一个成员
    */
    setup_per_cpu_areas();
    /*
    	该函数初始化多核处理器系统中的处理器位码表
    */
    smp_prepare_boot_cpu(); /* arch-specific boot-cpu hooks */
    boot_cpu_hotplug_init();

    /*
    	https://blog.csdn.net/liuhangtiant/article/details/80957313
    	这里调用的时候系统处于SYSTEM_BOOTING状态，系统有全局的system_state变量代表当前系统的状态
    	对于NUMA架构，每个节点的本地zone和remote zone都挂载到zonelist中
    	内部会调用build_all_zonelists_init
    */
    build_all_zonelists(NULL);
    // 热插拔cpu使用
    page_alloc_init();

    pr_notice("Kernel command line: %s\n", boot_command_line);
    /* parameters may set static keys */
    jump_label_init();
    parse_early_param();
    after_dashes = parse_args("Booting kernel",
                  static_command_line, __start___param,
                  __stop___param - __start___param,
                  -1, -1, NULL, &unknown_bootoption);
    if (!IS_ERR_OR_NULL(after_dashes))
        parse_args("Setting init args", after_dashes, NULL, 0, -1, -1,
               NULL, set_init_arg);

    /*
     * These use large bootmem allocations and must precede
     * kmem_cache_init()
     */
    // printk buf
    setup_log_buf(0);
    pidhash_init();
    // vfs的inode和dentry cache初始化
    vfs_caches_init_early();
    sort_main_extable();
    trap_init();
    /*
    	初始化内存分配器：buddy、sla|u|ob
    */
    mm_init();

    ftrace_init();

    /* trace_printk can be enabled here */
    early_trace_init();

    /*
     * Set up the scheduler prior starting any interrupts (such as the
     * timer interrupt). Full topology setup happens at smp_init()
     * time - but meanwhile we still have a functioning scheduler.
     */
    sched_init();
    /*
     * Disable preemption - early bootup scheduling is extremely
     * fragile until we cpu_idle() for the first time.
     */
    preempt_disable();
    if (WARN(!irqs_disabled(),
         "Interrupts were enabled *very* early, fixing it\n"))
        local_irq_disable();
    radix_tree_init();

    /*
     * Allow workqueue creation and work item queueing/cancelling
     * early.  Work item execution depends on kthreads and starts after
     * workqueue_init().
     */
    workqueue_init_early();

    rcu_init();
    
    /* Trace events are available after this */
    trace_init();

    context_tracking_init();
    /* init some links before init_ISA_irqs() */
    early_irq_init();
    init_IRQ();
    tick_init();
    rcu_init_nohz();
    init_timers();
    // 高精度定时器初始化
    hrtimers_init();
    softirq_init();
    timekeeping_init();
    time_init();
    sched_clock_postinit();
    printk_safe_init();
    perf_event_init();
    profile_init();
    call_function_init();
    WARN(!irqs_disabled(), "Interrupts were enabled early\n");
    early_boot_irqs_disabled = false;
    // 使能本地中断
    local_irq_enable();

    kmem_cache_init_late();

    /*
     * HACK ALERT! This is early. We're enabling the console before
     * we've done PCI setups etc, and console_init() must be aware of
     * this. But we do want output early, in case something goes wrong.
     */
    console_init();
    if (panic_later)
        panic("Too many boot %s vars at `%s'", panic_later,
              panic_param);

    lockdep_info();

    /*
     * Need to run this when irqs are enabled, because it wants
     * to self-test [hard/soft]-irqs on/off lock inversion bugs
     * too:
     */
    locking_selftest();
    
    /*
     * This needs to be called before any devices perform DMA
     * operations that might use the SWIOTLB bounce buffers. It will
     * mark the bounce buffers as decrypted so that their usage will
     * not cause "plain-text" data to be decrypted when accessed.
     */
    mem_encrypt_init();

#ifdef CONFIG_BLK_DEV_INITRD
    if (initrd_start && !initrd_below_start_ok &&
        page_to_pfn(virt_to_page((void *)initrd_start)) < min_low_pfn) {
        pr_crit("initrd overwritten (0x%08lx < 0x%08lx) - disabling it.\n",
            page_to_pfn(virt_to_page((void *)initrd_start)),
            min_low_pfn);
        initrd_start = 0;
    }
#endif
    // 内核内存泄露检测，需要打开config
    kmemleak_init();
    debug_objects_mem_init();
    setup_per_cpu_pageset();
    numa_policy_init();
    if (late_time_init)
        late_time_init();
    calibrate_delay();
    pidmap_init();
    anon_vma_init();
    acpi_early_init();
#ifdef CONFIG_X86
    if (efi_enabled(EFI_RUNTIME_SERVICES))
        efi_enter_virtual_mode();
#endif
    thread_stack_cache_init();
    cred_init();
    fork_init();
    proc_caches_init();
    buffer_init();
    key_init();
    security_init();
    dbg_late_init();
    /*
    	很重要的函数
    	
    */
    vfs_caches_init();
    pagecache_init();
    signals_init();
    proc_root_init();
    nsfs_init();
    cpuset_init();
    cgroup_init();
    taskstats_init_early();
    delayacct_init();

    check_bugs();

    acpi_subsystem_init();
    arch_post_acpi_subsys_init();
    sfi_init_late();
    if (efi_enabled(EFI_RUNTIME_SERVICES)) {
        efi_free_boot_services();
    }

    /* Do the rest non-__init'ed, we're now alive */
    rest_init();
}
```

