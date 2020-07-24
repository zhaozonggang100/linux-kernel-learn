### 1、概述

workqueue的初始化，在前面rest_init中已经介绍了内核模块的初始化流程，init_workqueue内核模块也是在那里初始化workqueue的，kernel_init_freeable-->do_basic_setup-->do_initcalls

### 2、源码注释

- 1、init_workqueue

```c
5459 static int __init init_workqueues(void)
5460 {
    	 // 普通进程的最高优先级nice：-20，prio：100
5461     int std_nice[NR_STD_WORKER_POOLS] = { 0, HIGHPRI_NICE_LEVEL };
5462     int i, cpu;
5463 
5464     WARN_ON(__alignof__(struct pool_workqueue) < __alignof__(long long));
5465 
5466     BUG_ON(!alloc_cpumask_var(&wq_unbound_cpumask, GFP_KERNEL));
5467     cpumask_copy(wq_unbound_cpumask, cpu_possible_mask);
5468 
    	 /*
    	 	创建pool_workqueue的工作队列线程池结构的slub缓存
    	 */
5469     pwq_cache = KMEM_CACHE(pool_workqueue, SLAB_PANIC);
5470 
    	 // 当smp_init之后会调用workqueue_cpu_up_callback
    	 // 跟随CPU_UP/CPU_DOWN动态创建工作线程的接口。
5471     cpu_notifier(workqueue_cpu_up_callback, CPU_PRI_WORKQUEUE_UP);
5472     hotcpu_notifier(workqueue_cpu_down_callback, CPU_PRI_WORKQUEUE_DOWN);
5473 
5474     wq_numa_init();
5475 
5476     /* initialize CPU pools */
    	 // 遍历每个possible状态的CPU
5477     for_each_possible_cpu(cpu) {
5478         struct worker_pool *pool;
5479 
5480         i = 0;
    		 // 每个CPU两个worker_poo，分别对应per-cpu变量cpu_worker_pool[0]和cpu_worker_pool[1]
5481         for_each_cpu_worker_pool(pool, cpu) {
    			 // 初始化worker_pool
5482             BUG_ON(init_worker_pool(pool));
5483             pool->cpu = cpu;
5484             cpumask_copy(pool->attrs->cpumask, cpumask_of(cpu));
5485             pool->attrs->nice = std_nice[i++]; // 设置nice值
5486             pool->node = cpu_to_node(cpu);
5487 
5488             /* alloc pool ID */
5489             mutex_lock(&wq_pool_mutex);
5490             BUG_ON(worker_pool_assign_id(pool));
5491             mutex_unlock(&wq_pool_mutex);
5492         }
5493     }
5494 
5495     /* create the initial worker */
5496     for_each_online_cpu(cpu) { // 遍历所有online状态CPU，对于SMP多核CPU，支队boot cpu创建了工作线程。其他CPU工作线程稍后再cpu_up中创建。
5497         struct worker_pool *pool;
5498 
5499         for_each_cpu_worker_pool(pool, cpu) { // 使用create_worker对每个worker_pool创建两个内核线程对应cpu_worker_pool[0]和cpu_worker_pool[1]
5500             pool->flags &= ~POOL_DISASSOCIATED;
5501             BUG_ON(!create_worker(pool));
5502         }
5503     }
5504 
5505     /* create default unbound and ordered wq attrs */
5506     for (i = 0; i < NR_STD_WORKER_POOLS; i++) {
5507         struct workqueue_attrs *attrs;  // 设置Unbound类型workqueue的属性
5508 
5509         BUG_ON(!(attrs = alloc_workqueue_attrs(GFP_KERNEL)));
5510         attrs->nice = std_nice[i];
5511         unbound_std_wq_attrs[i] = attrs; // 设置ordered类型workqueue的属性，ordered类型workqueue同一时刻只能有一个work item在运行
5512 
5513         /*
5514          * An ordered wq should have only one pwq as ordering is
5515          * guaranteed by max_active which is enforced by pwqs.
5516          * Turn off NUMA so that dfl_pwq is used for all nodes.
5517          */
5518         BUG_ON(!(attrs = alloc_workqueue_attrs(GFP_KERNEL)));
5519         attrs->nice = std_nice[i];
5520         attrs->no_numa = true;
5521         ordered_wq_attrs[i] = attrs;
5522     }
5523 
5524     system_wq = alloc_workqueue("events", 0, 0);  // 普通优先级bound类型工作队列system_wq
5525     system_highpri_wq = alloc_workqueue("events_highpri", WQ_HIGHPRI, 0);  // 高优先级bound类型工作队列system_highpri_wq
5526     system_long_wq = alloc_workqueue("events_long", 0, 0);
5527     system_unbound_wq = alloc_workqueue("events_unbound", WQ_UNBOUND,
5528                         WQ_UNBOUND_MAX_ACTIVE); // 普通优先级unbound类型工作队列system_unbound_wq
5529     system_freezable_wq = alloc_workqueue("events_freezable",
5530                           WQ_FREEZABLE, 0); // freezable类型工作队列system_freezable_wq
5531     system_power_efficient_wq = alloc_workqueue("events_power_efficient",
5532                           WQ_POWER_EFFICIENT, 0); // 省电类型的工作队列system_power_efficient_wq
5533     system_freezable_power_efficient_wq = alloc_workqueue("events_freezable_power_efficient",
5534                           WQ_FREEZABLE | WQ_POWER_EFFICIENT,
5535                           0); // freezable并且省电类型的工作队列system_freezable_power_efficient_wq
5536     BUG_ON(!system_wq || !system_highpri_wq || !system_long_wq ||
5537            !system_unbound_wq || !system_freezable_wq ||
5538            !system_power_efficient_wq ||
5539            !system_freezable_power_efficient_wq);
5540 
5541     wq_watchdog_init();
5542 
5543     return 0;
5544 }

// 注册内核模块初始化函数
5545 early_initcall(init_workqueues);
```



