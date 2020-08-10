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
    	函数(查看定义)给每个CPU分配内存，并拷贝.data.percpu段的数据. 为系统中的每个CPU的per_cpu变量申请空间. 
在SMP系统中, setup_per_cpu_areas初始化源代码中(使用per_cpu宏)定义的静态per-cpu变量, 这种变量对系统中每个CPU都有一个独立的副本. 
此类变量保存在内核二进制影像的一个独立的段中, setup_per_cpu_areas的目的就是为系统中各个CPU分别创建一份这些数据的副本
在非SMP系统中这是一个空操作
    */
    setup_per_cpu_areas();
    /*
    	该函数初始化多核处理器系统中的处理器位码表
    */
    smp_prepare_boot_cpu(); /* arch-specific boot-cpu hooks */
    boot_cpu_hotplug_init();

    /*
    	为系统中的zone建立后备zone的列表.
    	所有zone的后备列表都在pglist_data->node_zonelists[0]中;
		期间也对per-CPU变量boot_pageset做了初始化. 
		建立并初始化结点和内存域的数据结构
		【memory】
    */
    build_all_zonelists(NULL);
    // 热插拔cpu使用的内存【memory】
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
    // 根据低端内存页数和散列度，分配hash空间，并赋予pid_hash
    pidhash_init();
    // vfs的inode和dentry cache初始化
    vfs_caches_init_early();
    sort_main_extable();
    trap_init();
    /*
    	初始化内存分配器：buddy、sla|u|ob
    	建立了内核的内存分配器, 其中通过mem_init停用bootmem分配器并迁移到实际的内存管理器
    	(比如伙伴系统)，然后调用kmem_cache_init函数初始化内部小块内存区的分配器
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

    /*
    	在kmem_cache_init之后, 完善分配器的缓存机制,　当前3个可用的内核内存分配器slab, slob, slub都会定义此函数
    */
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
    /*
    	Kmemleak工作于内核态，Kmemleak 提供了一种可选的内核泄漏检测，其方法类似于跟踪内存收集器。当独立的对象没有被释放时，其报告记录在 /sys/kernel/debug/kmemleak中, Kmemcheck能够帮助定位大多数内存错误的上下文
    */
    kmemleak_init();
    debug_objects_mem_init();
    /*
    	初始化CPU高速缓存行, 为pagesets的第一个数组元素分配内存, 换句话说, 其实就是第一个
    	系统处理器分配由于在分页情况下，每次存储器访问都要存取多级页表，这就大大降低了访问速
    	度。所以，为了提高速度，在CPU中设置一个最近存取页面的高速缓存硬件机制，当进行存储器访
    	问时，先检查要访问的页面是否在高速缓存中
    */
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

### 2、setup_arch

```c
272 void __init setup_arch(char **cmdline_p)
273 {
274     init_mm.start_code = (unsigned long) _text;
275     init_mm.end_code   = (unsigned long) _etext;
276     init_mm.end_data   = (unsigned long) _edata;
277     init_mm.brk    = (unsigned long) _end;
278 
279     *cmdline_p = boot_command_line;
280 
281     early_fixmap_init();
282     early_ioremap_init();
283 
    	/*
    		1、从dts获取memory的信息（address-size对）
    	*/
284     setup_machine_fdt(__fdt_pointer);
285 
286     /*
287      * Initialise the static keys early as they may be enabled by the
288      * cpufeature code and early parameters.
289      */
290     jump_label_init();
291     parse_early_param();
292 
293     /*
294      * Unmask asynchronous aborts and fiq after bringing up possible
295      * earlycon. (Report possible System Errors once we can report this
296      * occurred).
297      */
298     local_daif_restore(DAIF_PROCCTX_NOIRQ);
299 
300     /*
301      * TTBR0 is only used for the identity mapping at this stage. Make it
302      * point to zero page to avoid speculatively fetching new entries.
303      */
304     cpu_uninstall_idmap();
305 
306     xen_early_init();
307     efi_init();
    
        /*
        	初始化memblock
        	
        	使用arm64_memblock_init来完成memblock机制的初始化工作, 至此memblock分配器
        	接受系统中系统中内存的分配工作
        */
308     arm64_memblock_init();
309 
    	/*
    		分页机制初始化
    		
    		调用paging_init来完成系统分页机制的初始化工作, 建立页表, 从而内核可以完成
    		虚拟地址的分配和转换
    	*/
310     paging_init();
311 
312     acpi_table_upgrade();
313 
314     /* Parse the ACPI tables for possible boot-time configuration */
315     acpi_boot_table_init();
316 
317     if (acpi_disabled)
318         unflatten_device_tree();
319 
    	/*
    		初始化内存管理
    		
    		最后调用bootmem_init来完成实现buddy内存管理所需要的工作
    		始化内存数据结构包括内存节点, 内存域和页帧page
    	*/
320     bootmem_init();
321 
322     kasan_init();
323 
324     request_standard_resources();
325 
326     early_ioremap_reset();
327 
328     if (acpi_disabled)
329         psci_dt_init();
330     else
331         psci_acpi_init();
332 
333     cpu_read_bootcpu_ops();
334     smp_init_cpus();
335     smp_build_mpidr_hash();
336 
337     /* Init percpu seeds for random tags after cpus are set up. */
338     kasan_init_tags();
339 
340 #ifdef CONFIG_ARM64_SW_TTBR0_PAN
341     /*
342      * Make sure init_thread_info.ttbr0 always generates translation
343      * faults in case uaccess_enable() is inadvertently called by the init
344      * thread.
345      */
346     init_task.thread_info.ttbr0 = __pa_symbol(empty_zero_page);
347 #endif
348 
349 #ifdef CONFIG_VT
350     conswitchp = &dummy_con;
351 #endif
352     if (boot_args[1] || boot_args[2] || boot_args[3]) {
353         pr_err("WARNING: x1-x3 nonzero in violation of boot protocol:\n"
354             "\tx1: %016llx\n\tx2: %016llx\n\tx3: %016llx\n"
355             "This indicates a broken bootloader or old kernel\n",
356             boot_args[1], boot_args[2], boot_args[3]);
357     }
358 }
```

