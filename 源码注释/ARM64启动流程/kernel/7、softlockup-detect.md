### 1、softlockup函数初始化流程

start_kernel-->rest_init--->kernel_init  1号线程创建--->kernel_init_freeable--->lockup_detector_init

### 2、函数详解

```c
// file：kernel/watchdog.c
// 参考：https://www.cnblogs.com/arnoldlu/p/10338850.html

 /*
 	watchdog/x线程创建之前需要做的工作
 */
 599 void watchdog_enable(unsigned int cpu)
 600 {   
         // 创建一个hrtimer，超时函数为watchdog_timer_fn，这里面会检查watchdog_touch_ts变量是否超过20秒没有被更新。如果是，则有soft lockup。
 601     struct hrtimer *hrtimer = raw_cpu_ptr(&watchdog_hrtimer);
 602     unsigned int *enabled = raw_cpu_ptr(&watchdog_en);
 603     
 604     if (*enabled)   
 605         return;     
 606     
 607     /* kick off the timer for the hardlockup detector */
 608     hrtimer_init(hrtimer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
 609     hrtimer->function = watchdog_timer_fn;
 610 
 611     /* Enable the perf event */
 612     watchdog_nmi_enable(cpu);
 613 
 614     /* done here because hrtimer_start can only pin to smp_processor_id() */
         // 启动一个超时为sample_period(4秒)的hrtimer，HRTIMER_MODE_REL_PINNED表示此hrtimer和当前CPU绑定。
 615     hrtimer_start(hrtimer, ns_to_ktime(sample_period),
 616               HRTIMER_MODE_REL_PINNED);
 617 
 618     /* initialize timestamp */
         // 设置当前线程为实时FIFO，并且优先级为实时99.这个优先级表示高于所有的非实时线程，但是实时优先级最低的。
 619     watchdog_set_prio(SCHED_FIFO, MAX_RT_PRIO - 1);
 620     __touch_watchdog(); // 更新watchdog_touch_ts变量，相当于喂狗操作。
 621 
 622     /* 
 623      * Need to ensure above operations are observed by other CPUs before
 624      * indicating that timer is enabled. This is to synchronize core
 625      * isolation and hotplug. Core isolation will wait for this flag to be
 626      * set.
 627      */ 
 628     mb();
 629     *enabled = 1;
 630 }
 
 /*
 	相当于watchdog_enable()反操作，将线程恢复为普通线程；取消hrtimer。
 */
 632 void watchdog_disable(unsigned int cpu)
 633 {
 634     struct hrtimer *hrtimer = raw_cpu_ptr(&watchdog_hrtimer);
 635     unsigned int *enabled = raw_cpu_ptr(&watchdog_en);
 636 
 637     if (!*enabled)
 638         return;
 639 
 640     watchdog_set_prio(SCHED_NORMAL, 0);
 641     hrtimer_cancel(hrtimer);
 642     /* disable the perf event */
 643     watchdog_nmi_disable(cpu);
 644 
 645     /*
 646      * No need for barrier here since disabling the watchdog is
 647      * synchronized with hotplug lock
 648      */
 649     *enabled = 0;
 650 }

 /*
 	hrtimer_interrupts记录了产生hrtimer的次数；在watchdog()中，将hrtimer_interrupts赋给soft_lockup_hrtimer_cnt。两者相等表示没有hrtimer产生，不需要运行watchdog/x线程；相反不等，则需要watchdog/x线程运行。
 */
 662 static int watchdog_should_run(unsigned int cpu)
 663 {   
 664     return __this_cpu_read(hrtimer_interrupts) !=
 665         __this_cpu_read(soft_lockup_hrtimer_cnt);
 666 }

 /*
 	
 */
 668 /*
 669  * The watchdog thread function - touches the timestamp.
 670  *
 671  * It only runs once every sample_period seconds (4 seconds by
 672  * default) to reset the softlockup timestamp. If this gets delayed
 673  * for more than 2*watchdog_thresh seconds then the debug-printout
 674  * triggers in watchdog_timer_fn().
 675  */
 676 static void watchdog(unsigned int cpu)
 677 {
     	 // 更新soft_lockup_hrtimer_cnt，在watch_should_run()中就返回false，表示线程不需要运行，即不需要喂狗。
 678     __this_cpu_write(soft_lockup_hrtimer_cnt,
 679              __this_cpu_read(hrtimer_interrupts));
 680     __touch_watchdog(); // 虽然就是一句话，但是却很重要的喂狗操作。
 681 
 682     /*
 683      * watchdog_nmi_enable() clears the NMI_WATCHDOG_ENABLED bit in the
 684      * failure path. Check for failures that can occur asynchronously -
 685      * for example, when CPUs are on-lined - and shut down the hardware
 686      * perf event on each CPU accordingly.
 687      *
 688      * The only non-obvious place this bit can be cleared is through
 689      * watchdog_nmi_enable(), so a pr_info() is placed there.  Placing a
 690      * pr_info here would be too noisy as it would result in a message
 691      * every few seconds if the hardlockup was disabled but the softlockup
 692      * enabled.
 693      */
 694     if (!(watchdog_enabled & NMI_WATCHDOG_ENABLED))
 695         watchdog_nmi_disable(cpu);
 696 }

836 static struct smp_hotplug_thread watchdog_threads = {
 837     .store          = &softlockup_watchdog,	// 存放percpu strcut task_strcut指针的指针
 838     .thread_should_run  = watchdog_should_run, // 检查是否应该运行watchdog/x线程。
 839     .thread_fn      = watchdog,				// 内核线程执行函数
 840     .thread_comm        = "watchdog/%u",		// 内核线程名字
 841     .setup          = watchdog_enable,       // 在运行watchdog/x线程之前的准备工作。
 842     .cleanup        = watchdog_cleanup,	// 在退出watchdog/x线程之后的清楚工作。
 843     .park           = watchdog_disable, // 当CPU offline时，需要临时停止。
 844     .unpark         = watchdog_enable,	// 当CPU变成online时，进行准备工作。
 845 };

 950 static int watchdog_enable_all_cpus(void)
 951 {   
 952     int err = 0;
 953     
     	 /*
     	 	如果当前watchdog线程没有运行，则下面创建每个cpu的watchdog线程（在cpumask的掩码范围内）,内核启动会走这里
     	 */
 954     if (!watchdog_running) {
 955         err = smpboot_register_percpu_thread_cpumask(&watchdog_threads,
 956                                  &watchdog_cpumask);
 957         if (err)
 958             pr_err("Failed to create watchdog threads, disabled\n");
 959         else
 960             watchdog_running = 1;
 961     } else {
 962         /*
 963          * Enable/disable the lockup detectors or
 964          * change the sample period 'on the fly'.
 965          */
 966         err = update_watchdog_all_cpus();
 967 
 968         if (err) {
 969             watchdog_disable_all_cpus();
 970             pr_err("Failed to update lockup detectors, disabled\n");
 971         }
 972     }
 973     
 974     if (err)
 975         watchdog_enabled = 0;
 976 
 977     return err;
 978 }
 980 static void watchdog_disable_all_cpus(void)
 981 {   
 982     if (watchdog_running) {
 983         watchdog_running = 0;
 984         smpboot_unregister_percpu_thread(&watchdog_threads);
 985     }
 986 }

1206 void __init lockup_detector_init(void)
1207 {   
    	 /*
    	 	获取变量sample_period，为watchdog_thresh*2/5，即4秒喂一次狗。
    	 */
1208     set_sample_period();
1209     
1210 #ifdef CONFIG_NO_HZ_FULL
1211     if (tick_nohz_full_enabled()) {
1212         pr_info("Disabling watchdog on nohz_full cores by default\n");
1213         cpumask_copy(&watchdog_cpumask, housekeeping_mask);
1214     } else
1215         cpumask_copy(&watchdog_cpumask, cpu_possible_mask);
1216 #else
1217     cpumask_copy(&watchdog_cpumask, cpu_possible_mask);
1218 #endif
1219         
    	 /*
    	 	如果当前watchdog_running没有再运行，那么为每个CPU创建一个watchdog/x线程，
    	 	这些线程每隔sample_period时间喂一次狗。watchdog_threads时watchdog/x线程
    	 	的主要输入参数，watchdog_cpumask规定了为哪些CPU创建线程
    	 */
1220     if (watchdog_enabled)
1221         watchdog_enable_all_cpus();
1222 }
```

