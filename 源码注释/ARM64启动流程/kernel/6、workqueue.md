### 1、概述

workqueue的初始化，在前面rest_init中已经介绍了内核模块的初始化流程，init_workqueue内核模块也是在那里初始化workqueue的，kernel_init_freeable-->do_basic_setup-->do_initcalls

### 2、源码注释

- 1、init_workqueue是内核启动时，会默认创建一些系统的工作线程，同时创建一些常用的workqueue（内核初始化调用）

```c
5459 static int __init init_workqueues(void)
5460 {
    	 // 普通进程的最高优先级nice：-20，prio：100
    	 // NR_STD_WORKER_POOLS = 2,每个cpu两个标准的worker_poll
5461     int std_nice[NR_STD_WORKER_POOLS] = { 0, HIGHPRI_NICE_LEVEL };
5462     int i, cpu;
5463 
5464     WARN_ON(__alignof__(struct pool_workqueue) < __alignof__(long long));
5465 
5466     BUG_ON(!alloc_cpumask_var(&wq_unbound_cpumask, GFP_KERNEL));
5467     cpumask_copy(wq_unbound_cpumask, cpu_possible_mask);
5468 
    	 // 创建pool_workqueue的工作队列线程池结构的slub缓存
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
    	 // 遍历每个possible状态的CPU，初始化每个cpu的worker pools
5477     for_each_possible_cpu(cpu) {
5478         struct worker_pool *pool;
5479 
5480         i = 0;
    		 // 每个CPU两个worker_pool，分别对应per-cpu变量cpu_worker_pool[0]和cpu_worker_pool[1]
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
    			 // 创建系统的工作线程并判断返回是否为NULL
5501             BUG_ON(!create_worker(pool));
5502         }
5503     }
5504 
5505     /* create default unbound and ordered wq attrs */
    	 /*
    	 	创建不绑定cpu的kworker的属性成员attrs
    	 */
5506     for (i = 0; i < NR_STD_WORKER_POOLS; i++) {
5507         struct workqueue_attrs *attrs;  // 设置Unbound类型workqueue的属性
5508 
5509         BUG_ON(!(attrs = alloc_workqueue_attrs(GFP_KERNEL)));
5510         attrs->nice = std_nice[i];		// 优先级为39，nice值-20
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
    	 // 普通优先级bound类型工作队列
5524     system_wq = alloc_workqueue("events", 0, 0);  system_wq
    	 // 高优先级bound类型工作队列system_highpri_wq
5525     system_highpri_wq = alloc_workqueue("events_highpri", WQ_HIGHPRI, 0);  
5526     system_long_wq = alloc_workqueue("events_long", 0, 0);
         // 普通优先级unbound类型工作队列system_unbound_wq
5527     system_unbound_wq = alloc_workqueue("events_unbound", WQ_UNBOUND,
5528                         WQ_UNBOUND_MAX_ACTIVE); 
    	 // freezable类型工作队列system_freezable_wq
5529     system_freezable_wq = alloc_workqueue("events_freezable",
5530                           WQ_FREEZABLE, 0); 
    	 // 省电类型的工作队列system_power_efficient_wq
5531     system_power_efficient_wq = alloc_workqueue("events_power_efficient",
5532                           WQ_POWER_EFFICIENT, 0); 
    	 // freezable并且省电类型的工作队列system_freezable_power_efficient_wq
5533     system_freezable_power_efficient_wq = alloc_workqueue("events_freezable_power_efficient",
5534                           WQ_FREEZABLE | WQ_POWER_EFFICIENT,
5535                           0); 
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

- 2、使用create_worker创建系统需要的工作线程（内核初始化调用）

```c
1728 /**
1729  * create_worker - create a new workqueue worker
1730  * @pool: pool the new worker will belong to
1731  *
1732  * Create and start a new worker which is attached to @pool.
1733  *  
1734  * CONTEXT:
1735  * Might sleep.  Does GFP_KERNEL allocations.
1736  *
1737  * Return:
1738  * Pointer to the newly created worker.
1739  */
1740 static struct worker *create_worker(struct worker_pool *pool)
1741 {   
1742     struct worker *worker = NULL;
1743     int id = -1;
1744     char id_buf[16];
1745 
1746     /* ID is needed to determine kthread name */
    	 // 从当前worker_pool->worker_ida获取一个空闲id
1747     id = ida_simple_get(&pool->worker_ida, 0, 0, GFP_KERNEL);
1748     if (id < 0)
1749         goto fail;
1750 
1751     worker = alloc_worker(pool->node);
1752     if (!worker)
1753         goto fail;
1754 
1755     worker->pool = pool; // 该woker属于参数worker_pool
1756     worker->id = id;	// 根据worker的线程名字计算的id作为wokrer的id
1757 
1758     if (pool->cpu >= 0)
1759         snprintf(id_buf, sizeof(id_buf), "%d:%d%s", pool->cpu, id,
1760              pool->attrs->nice < 0  ? "H" : "");
1761     else
1762         snprintf(id_buf, sizeof(id_buf), "u%d:%d", pool->id, id);
1763     
    	 /*
    	 	根据id_buf的值创建内核线程取对应的id_buf的名字，buf_id解释如下
    	 		kworker/xx:xxH			bound cpu的kworker，高优先级级，nice < 0（-20）
    	 		kworker/xx:xx			bound cpu的kworker，普通优先级，nice > 0
    	 		cpuid:worker->id
    	 		
    	 		kworker/uxx:xx			非bound cpu的kworker（u pool-id: worker->id）
    	 */
1764     worker->task = kthread_create_on_node(worker_thread, worker, pool->node,
1765                           "kworker/%s", id_buf);
1766     if (IS_ERR(worker->task))
1767         goto fail;
1768 
1769     set_user_nice(worker->task, pool->attrs->nice);	// 设置worker对应task的nice
1770     kthread_bind_mask(worker->task, pool->attrs->cpumask); // 根据bound属性的cpumask设置线程是否bound cpu
1771 
1772     /* successful, attach the worker to the pool */
    	 // 将worker加入到线程池管理，也就是在woker的node成员双向链表头尾插pool_worker结构，代表该
    	 // worker归pool_worker管理
1773     worker_attach_to_pool(worker, pool);
1774 
1775     /* start the newly created worker */
1776     spin_lock_irq(&pool->lock);
1777     worker->pool->nr_workers++; // pool管理的worker+1
1778     worker_enter_idle(worker);	// 设置worker为idle状态
1779     wake_up_process(worker->task); // 唤醒刚创建的worker
1780     spin_unlock_irq(&pool->lock);
1781 
1782     return worker;	// 返回创建的worker
1783 
1784 fail:
1785     if (id >= 0)
1786         ida_simple_remove(&pool->worker_ida, id);
1787     kfree(worker);
1788     return NULL;
1789 }
```

- 3、常用work queue 接口

```c
/*
	将work放入system_wq（普通优先级bound cpu的全局workqueue）中，最后由非绑定cpu的kworker执行
*/
531 static inline bool schedule_work(struct work_struct *work)
532 {
533     return queue_work(system_wq, work);
534 }

/*
	相比上面的api，将work放入system_wq，并指定最终被哪个普通优先级的kworker调用
*/
515 static inline bool schedule_work_on(int cpu, struct work_struct *work)
516 {
517     return queue_work_on(cpu, system_wq, work);
518 }

/*
	将work放入指定的工作队列（可以自定义），该工作队列不bound  cpu
*/
472 static inline bool queue_work(struct workqueue_struct *wq,
473                   struct work_struct *work)
474 {   
475     return queue_work_on(WORK_CPU_UNBOUND, wq, work);
476 }

/*
	等到延时时间到之后将work加入指定的工作队列，工作队列非bound cpu
*/
486 static inline bool queue_delayed_work(struct workqueue_struct *wq,
487                       struct delayed_work *dwork,
488                       unsigned long delay)
489 {   
490     return queue_delayed_work_on(WORK_CPU_UNBOUND, wq, dwork, delay);
491 }

/*
	创建工作队列，工作队列中的work顺序执行，同一时间最多只能执行一个
*/
411 #define alloc_ordered_workqueue(fmt, flags, args...)            \
412     alloc_workqueue(fmt, WQ_UNBOUND | __WQ_ORDERED |        \
413             __WQ_ORDERED_EXPLICIT | (flags), 1, ##args)
    
/*
	创建工作队列，用于内存不足的场景，这样创建的workqueue最终被rescuer_thread接管
	创建的工作队列中的work是顺序执行的，不能并发
*/
420 #define create_singlethread_workqueue(name)             \
421     alloc_ordered_workqueue("%s", WQ_MEM_RECLAIM, name)
    
/*
	创建工作队列，创建的工作队列中的wokr在执行时发生suspend，需要等待workqueue中当前执行的work
	执行结束然后suspend
*/
417 #define create_freezable_workqueue(name)                \
418     alloc_workqueue("%s", WQ_FREEZABLE | WQ_UNBOUND | WQ_MEM_RECLAIM, \
419             1, (name))

/*
	创建工作队列用于内存不足的场景，避免因分配内存失败导致创建线程失败
*/
415 #define create_workqueue(name)                      \
416     alloc_workqueue("%s", WQ_MEM_RECLAIM, 1, (name))
```

- 4、创建workqueue结构低层api

```c
3846 struct workqueue_struct *__alloc_workqueue_key(const char *fmt,
3847                            unsigned int flags,
3848                            int max_active,
3849                            struct lock_class_key *key,
3850                            const char *lock_name, ...)
3851 {
3852     size_t tbl_size = 0;
3853     va_list args;
3854     struct workqueue_struct *wq;
3855     struct pool_workqueue *pwq;
3856 
3857     /*
3858      * Unbound && max_active == 1 used to imply ordered, which is no
3859      * longer the case on NUMA machines due to per-node pools.  While
3860      * alloc_ordered_workqueue() is the right way to create an ordered
3861      * workqueue, keep the previous behavior to avoid subtle breakages
3862      * on NUMA.
3863      */
3864     if ((flags & WQ_UNBOUND) && max_active == 1)
3865         flags |= __WQ_ORDERED;
3866 
3867     /* see the comment above the definition of WQ_POWER_EFFICIENT */
3868     if ((flags & WQ_POWER_EFFICIENT) && wq_power_efficient)
3869         flags |= WQ_UNBOUND;
3870 
3871     /* allocate wq and format name */
3872     if (flags & WQ_UNBOUND)
3873         tbl_size = nr_node_ids * sizeof(wq->numa_pwq_tbl[0]);
3874 
3875     wq = kzalloc(sizeof(*wq) + tbl_size, GFP_KERNEL);
3876     if (!wq)
3877         return NULL;
3878 
3879     if (flags & WQ_UNBOUND) {
3880         wq->unbound_attrs = alloc_workqueue_attrs(GFP_KERNEL);
3881         if (!wq->unbound_attrs)
3882             goto err_free_wq;
3883     }
3884 
3885     va_start(args, lock_name); 
3886     vsnprintf(wq->name, sizeof(wq->name), fmt, args);
3887     va_end(args);
3888 
    	 /*
    	 	WQ_DFL_ACTIVE = 256，一个工作队列中处于active的work的最大数量
    	 	一般创建的工作队列最多有一个active的work，自己调用allco_workqueue可以传入自定的最大数
    	 */
3889     max_active = max_active ?: WQ_DFL_ACTIVE;
3890     max_active = wq_clamp_max_active(max_active, flags, wq->name);
3891 
3892     /* init wq */
3893     wq->flags = flags;
3894     wq->saved_max_active = max_active;
3895     mutex_init(&wq->mutex);
3896     atomic_set(&wq->nr_pwqs_to_flush, 0);
3897     INIT_LIST_HEAD(&wq->pwqs);
3898     INIT_LIST_HEAD(&wq->flusher_queue);
3899     INIT_LIST_HEAD(&wq->flusher_overflow);
3900     INIT_LIST_HEAD(&wq->maydays);
3901 
3902     lockdep_init_map(&wq->lockdep_map, lock_name, key, 0);
3903     INIT_LIST_HEAD(&wq->list);
3904 
3905     if (alloc_and_link_pwqs(wq) < 0)
3906         goto err_free_wq;
3907 
3908     /*
3909      * Workqueues which may be used during memory reclaim should
3910      * have a rescuer to guarantee forward progress.
3911      */
    	 /*
    	 	如果设置了WQ_MEM_RECLAIM标志，会创建一个内核线程，线程处理函数是rescuer_thread
    	 */
3912     if (flags & WQ_MEM_RECLAIM) {
3913         struct worker *rescuer;
3914 
3915         rescuer = alloc_worker(NUMA_NO_NODE);
3916         if (!rescuer)
3917             goto err_destroy;
3918 
3919         rescuer->rescue_wq = wq;
3920         rescuer->task = kthread_create(rescuer_thread, rescuer, "%s",
3921                            wq->name);
3922         if (IS_ERR(rescuer->task)) {
3923             kfree(rescuer);
3924             goto err_destroy;
3925         }
3926 		
    		 // workqueue的rescuer成员设置了，说明该workqueue由rescuer_thread从workqueue中取work并执行work对应的function
3927         wq->rescuer = rescuer;
    		 // 设置创建的内核线程的cpu亲和性
3928         kthread_bind_mask(rescuer->task, cpu_possible_mask);
    		 // 唤醒刚创建的rescuer_thread内核线程
3929         wake_up_process(rescuer->task);
3930     }
3931 
    	 /*
    	 	其余的workqueue交由kworker处理
    	 */
3932     if ((wq->flags & WQ_SYSFS) && workqueue_sysfs_register(wq))
3933         goto err_destroy;
3934 
3935     /*
3936      * wq_pool_mutex protects global freeze state and workqueues list.
3937      * Grab it, adjust max_active and add the new @wq to workqueues
3938      * list.
3939      */
3940     mutex_lock(&wq_pool_mutex);
3941 
3942     mutex_lock(&wq->mutex);
3943     for_each_pwq(pwq, wq)
3944         pwq_adjust_max_active(pwq);
3945     mutex_unlock(&wq->mutex);
3946 
3947     list_add_tail_rcu(&wq->list, &workqueues);
3948 
3949     mutex_unlock(&wq_pool_mutex);
3950 
3951     return wq;
3952 
3953 err_free_wq:
3954     free_workqueue_attrs(wq->unbound_attrs);
3955     kfree(wq);
3956     return NULL;
3957 err_destroy:
3958     destroy_workqueue(wq);
3959     return NULL;
3960 }
3961 EXPORT_SYMBOL_GPL(__alloc_workqueue_key);
```



### 3、数据结构

- 1、工作结构

```c
// 普通工作结构  
100 struct work_struct {
    	/*
    		低比特位部分是work的标志位，剩余比特位通常用于存放上一次运行的worker_pool ID或pool_workqueue的指针。存放的内容有WORK_STRUCT_PWQ标志位来决定
    	*/
101     atomic_long_t data;	
102     struct list_head entry;		// 挂载work的队列
103     work_func_t func;			// 该work的处理函数
104 #ifdef CONFIG_LOCKDEP
105     struct lockdep_map lockdep_map;
106 #endif
107 };

// 延时工作结构
struct delayed_work {
	struct work_sturct work;		// 普通工作结构
	struct timer_list timer;		// 定时器链表
	struct workqueue_struct *wq;	// 工作队列结构
	int cpu;						// 是否bound cpu
}
```



- 2、工作队列结构

```c
 233 /*
 234  * The externally visible workqueue.  It relays the issued work items to
 235  * the appropriate worker_pool through its pool_workqueues.
 236  */
 237 struct workqueue_struct {
     	 // 该workqueue所在的所有pool_workqueue链表
 238     struct list_head    pwqs;       /* WR: all pwqs of this wq */
     	 // 系统所有workqueue_struct的全局链表
 239     struct list_head    list;       /* PR: list of all workqueues */
 240 
 241     struct mutex        mutex;      /* protects this wq */
 242     int         work_color; /* WQ: current work color */
 243     int         flush_color;    /* WQ: current flush color */
 244     atomic_t        nr_pwqs_to_flush; /* flush in progress */
 245     struct wq_flusher   *first_flusher; /* WQ: first flusher */
 246     struct list_head    flusher_queue;  /* WQ: flush waiters */
 247     struct list_head    flusher_overflow; /* WQ: flush overflow list */
 248 
     	 // 所有rescue状态下的pool_workqueue数据结构链表
 249     struct list_head    maydays;    /* MD: pwqs requesting rescue */
     	 // rescue内核线程，内存紧张时创建新的工作线程可能会失败，如果创建workqueue是设置了WQ_MEM_RECLAIM，那么rescuer线程会接管这种情况。
 250     struct worker       *rescuer;   /* I: rescue worker */
 251 
 252     int         nr_drainers;    /* WQ: drain in progress */
 253     int         saved_max_active; /* WQ: saved pwq max_active */
 254     
     	 // UNBOUND类型属性
 255     struct workqueue_attrs  *unbound_attrs; /* PW: only for unbound wqs */
     	 // unbound类型的pool_workqueue
 256     struct pool_workqueue   *dfl_pwq;   /* PW: only for unbound wqs */
 257     
 258 #ifdef CONFIG_SYSFS
 259     struct wq_device    *wq_dev;    /* I: for sysfs interface */
 260 #endif
 261 #ifdef CONFIG_LOCKDEP
 262     struct lockdep_map  lockdep_map;
 263 #endif
     	 // 该workqueue的名字
 264     char            name[WQ_NAME_LEN]; /* I: workqueue name */
 265     
 266     /*
 267      * Destruction of workqueue_struct is sched-RCU protected to allow
 268      * walking the workqueues list without grabbing wq_pool_mutex.
 269      * This is used to dump all workqueues from sysrq.
 270      */
 271     struct rcu_head     rcu;
 272 
     	 // 经常被不同CUP访问，因此要和cache line对齐
 273     /* hot fields used during command issue, aligned to cacheline */
 274     unsigned int        flags ____cacheline_aligned; /* WQ: WQ_* flags */
     	 // 指向per-cpu类型的pool_workqueue
 275     struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */
 276     struct pool_workqueue __rcu *numa_pwq_tbl[]; /* PWR: unbound pwqs indexed by node */
 277 };
```



- 3、工作者线程（worker），运行work_struct

```c
 16 /*
 17  * The poor guys doing the actual heavy lifting.  All on-duty workers are
 18  * either serving the manager role, on idle list or on busy hash.  For
 19  * details on the locking annotation (L, I, X...), refer to workqueue.c.
 20  *
 21  * Only to be used in workqueue and async.
 22  */
 23 struct worker {
 24     /* on idle list while idle, on busy hash table while busy */
 25     union {
 26         struct list_head    entry;  /* L: while idle */
 27         struct hlist_node   hentry; /* L: while busy */
 28     };
 29 
     	// 当前正在处理的work
 30     struct work_struct  *current_work;  /* L: work being processed */
     	// 当前正在执行的work回调函数
 31     work_func_t     current_func;   /* L: current_work's fn */
     	// 当前work所属的pool_workqueue
 32     struct pool_workqueue   *current_pwq; /* L: current_work's pwq */
 33     bool            desc_valid; /* ->desc is valid */
     	// 所有被调度并正准备执行的work_struct都挂入该链表中
 34     struct list_head    scheduled;  /* L: scheduled works */
 35 
 36     /* 64 bytes boundary on 64bit, 32 on 32bit */
 37 
     	// 该工作线程的task_struct数据结构
 38     struct task_struct  *task;      /* I: worker task */
     	// 该工作线程所属的worker_pool
 39     struct worker_pool  *pool;      /* I: the associated pool */
 40                         /* L: for rescuers */
     	// 可以把该worker挂入到worker_pool->workers链表中
 41     struct list_head    node;       /* A: anchored at pool->workers */
 42                         /* A: runs through worker->node */
 43 
 44     unsigned long       last_active;    /* L: last active timestamp */
 45     unsigned int        flags;      /* X: flags */
 46     int         id;     /* I: worker id */
 47 
 48     /*
 49      * Opaque string set with work_set_desc().  Printed out with task
 50      * dump for debugging - WARN, BUG, panic or sysrq.
 51      */
 52     char            desc[WORKER_DESC_LEN];
 53 
 54     /* used only by rescuers to point to the target workqueue */
 55     struct workqueue_struct *rescue_wq; /* I: the workqueue to rescue */
 56 };
```



- 4、CMWQ提出了工作线程池的概念，工作线程池

  worker_pool是per-cpu变量，每个CPU都有worker_pool，而且有两个worker_pool。

  一个用于普通优先级工作线程，另一个用于高优先级工作线程。

```c
 109 /*
 110  * Structure fields follow one of the following exclusion rules.
 111  *
 112  * I: Modifiable by initialization/destruction paths and read-only for
 113  *    everyone else.
 114  *
 115  * P: Preemption protected.  Disabling preemption is enough and should
 116  *    only be modified and accessed from the local cpu.
 117  *
 118  * L: pool->lock protected.  Access with pool->lock held.
 119  *
 120  * X: During normal operation, modification requires pool->lock and should
 121  *    be done only from local cpu.  Either disabling preemption on local
 122  *    cpu or grabbing pool->lock is enough for read access.  If
 123  *    POOL_DISASSOCIATED is set, it's identical to L.
 124  *
 125  * A: pool->attach_mutex protected.
 126  *
 127  * PL: wq_pool_mutex protected.
 128  *
 129  * PR: wq_pool_mutex protected for writes.  Sched-RCU protected for reads.
 130  *
 131  * PW: wq_pool_mutex and wq->mutex protected for writes.  Either for reads.
 132  *
 133  * PWR: wq_pool_mutex and wq->mutex protected for writes.  Either or
 134  *      sched-RCU for reads.
 135  *
 136  * WQ: wq->mutex protected.
 137  *
 138  * WR: wq->mutex protected for writes.  Sched-RCU protected for reads.
 139  *
 140  * MD: wq_mayday_lock protected.
 141  */
 142 
 143 /* struct worker is defined in workqueue_internal.h */
 144
  145 struct worker_pool {
 146     spinlock_t      lock;       /* the pool lock */
      	 // 对于unbound类型为-1；对于bound类型workqueue表示绑定的CPU ID
 147     int         cpu;        /* I: the associated cpu */
 148     int         node;       /* I: the associated node ID */
      	 // 该worker_pool的ID号
 149     int         id;     /* I: pool ID */
 150     unsigned int        flags;      /* X: flags */
 151 
     	 // 挂入pending状态的work_struct
 152     struct list_head    worklist;   /* L: list of pending works */
      	 // 工作线程的数量
 153     int         nr_workers; /* L: total number of workers */
 154 
     	 // 处于idle状态的工作线程的数量
 155     /* nr_idle includes the ones off idle_list for rebinding */
 156     int         nr_idle;    /* L: currently idle ones */
 157 
     	 // 处于idle状态的工作线程链表
 158     struct list_head    idle_list;  /* X: list of idle workers */
 159     struct timer_list   idle_timer; /* L: worker idle timeout */
 160     struct timer_list   mayday_timer;   /* L: SOS timer for workers */
 161 
 162     /* a workers is either on busy_hash or idle_list, or the manager */
 163     DECLARE_HASHTABLE(busy_hash, BUSY_WORKER_HASH_ORDER);
 164                         /* L: hash of busy workers */
 165 
 166     /* see manage_workers() for details on the two manager mutexes */
 167     struct worker       *manager;   /* L: purely informational */
 168     struct mutex        attach_mutex;   /* attach/detach exclusion */
      	 // 该worker_pool管理的工作线程链表
 169     struct list_head    workers;    /* A: attached workers */
 170     struct completion   *detach_completion; /* all workers detached */
 171 
 172     struct ida      worker_ida; /* worker IDs for task name */
 173 
     	 // 工作线程属性
 174     struct workqueue_attrs  *attrs;     /* I: worker attributes */
 175     struct hlist_node   hash_node;  /* PL: unbound_pool_hash node */
 176     int         refcnt;     /* PL: refcnt for unbound pools */
 177 
 178     /*
 179      * The current concurrency level.  As it's likely to be accessed
 180      * from other CPUs during try_to_wake_up(), put it in a separate
 181      * cacheline.
 182      */
     	 // 用于管理worker的创建和销毁的统计计数，表示运行中的worker数量。该变量可能被多CPU同时访问，因此独占一个缓存行，避免多核读写造成“颠簸”现象。
 183     atomic_t        nr_running ____cacheline_aligned_in_smp;
 184 
 185     /*
 186      * Destruction of pool is sched-RCU protected to allow dereferences
 187      * from get_work_pool().
 188      */
 189     struct rcu_head     rcu;
 190 } ____cacheline_aligned_in_smp;
```



- 5、连接workqueue和work-pool

```c
 192 /*
 193  * The per-pool workqueue.  While queued, the lower WORK_STRUCT_FLAG_BITS
 194  * of work_struct->data are used for flags and the remaining high bits
 195  * point to the pwq; thus, pwqs need to be aligned at two's power of the
 196  * number of flag bits.
 197  */
 198 struct pool_workqueue {
     	 // 指向worker_pool结构
 199     struct worker_pool  *pool;      /* I: the associated pool */
     	 // 指向workqueue_struct结构
 200   	 struct workqueue_struct *wq;        /* I: the owning workqueue */
 201     int         work_color; /* L: current color */
 202     int         flush_color;    /* L: flushing color */
 203     int         refcnt;     /* L: reference count */
 204     int         nr_in_flight[WORK_NR_COLORS];
 205                         /* L: nr of in_flight works */
     	 // 活跃的work_strcut数量
 206     int         nr_active;  /* L: nr of active works */
     	 // 最大活跃work_struct数量
 207     int         max_active; /* L: max active works */
     	 // 延迟执行work_struct链表
 208     struct list_head    delayed_works;  /* L: delayed works */
 209     struct list_head    pwqs_node;  /* WR: node on wq->pwqs */
 210     struct list_head    mayday_node;    /* MD: node on wq->maydays */
 211 
 212     /*
 213      * Release of unbound pwq is punted to system_wq.  See put_pwq()
 214      * and pwq_unbound_release_workfn() for details.  pool_workqueue
 215      * itself is also sched-RCU protected so that the first pwq can be
 216      * determined without grabbing wq->mutex.
 217      */
 218     struct work_struct  unbound_release_work;
 219     struct rcu_head     rcu;
 220 } __aligned(1 << WORK_STRUCT_FLAG_BITS);
```



- 6、影响创建workqueue的枚举flags

```c
273 /*
274  * Workqueue flags and constants.  For details, please refer to
275  * Documentation/workqueue.txt.
276  */
277 enum {
    	// workqueue不绑定到cpu
278     WQ_UNBOUND      = 1 << 1, /* not bound to any cpu */
    	// 在suspend进行进程冻结的时候，需要让工作线程完成当前所有的work才完成进程冻结，并且这个过程不会再新开始一个work的执行，直到进程被解冻。
279     WQ_FREEZABLE        = 1 << 2, /* freeze during suspend */
    	// 在内存紧张导致创建新进程失败，系统通过rescuer内核线程去接管这种情况。
280     WQ_MEM_RECLAIM      = 1 << 3, /* may be used for memory reclaim */
    	// 属于高于高优先级的worker_pool
281     WQ_HIGHPRI      = 1 << 4, /* high priority */
    	// 属于特别消耗CPU资源的一类work，这个work执行会得到调度器的监管，排在这类work后的non-CPU-intensive类型work可能会推迟执行
282     WQ_CPU_INTENSIVE    = 1 << 5, /* cpu intensive workqueue */
283     WQ_SYSFS        = 1 << 6, /* visible in sysfs, see wq_sysfs_register() */
284 
285     /* 
286      * Per-cpu workqueues are generally preferred because they tend to
287      * show better performance thanks to cache locality.  Per-cpu
288      * workqueues exclude the scheduler from choosing the CPU to
289      * execute the worker threads, which has an unfortunate side effect
290      * of increasing power consumption.
291      * 
292      * The scheduler considers a CPU idle if it doesn't have any task
293      * to execute and tries to keep idle cores idle to conserve power;
294      * however, for example, a per-cpu work item scheduled from an
295      * interrupt handler on an idle CPU will force the scheduler to
296      * excute the work item on that CPU breaking the idleness, which in
297      * turn may lead to more scheduling choices which are sub-optimal
298      * in terms of power consumption.
299      * 
300      * Workqueues marked with WQ_POWER_EFFICIENT are per-cpu by default
301      * but become unbound if workqueue.power_efficient kernel param is
302      * specified.  Per-cpu workqueues which are identified to
303      * contribute significantly to power-consumption are identified and
304      * marked with this flag and enabling the power_efficient mode
305      * leads to noticeable power saving at the cost of small
306      * performance disadvantage.
307      *
308      * http://thread.gmane.org/gmane.linux.kernel/1480396
309      */
    	// 根据wq_power_efficient来决定此类型的工作队列是bound还是unbound类型，bound型可能导致处于idle的CPU被唤醒，而unbound型则不会必然唤醒idle的CPU。
310     WQ_POWER_EFFICIENT  = 1 << 7,
311 
312     __WQ_DRAINING       = 1 << 16, /* internal: workqueue is draining */
    	// 同一个workqueue中的work是顺序执行的
313     __WQ_ORDERED        = 1 << 17, /* internal: workqueue is ordered */
314     __WQ_ORDERED_EXPLICIT   = 1 << 19, /* internal: alloc_ordered_workqueue() */
315 
316     WQ_MAX_ACTIVE       = 512,    /* I like 512, better ideas? */
317     WQ_MAX_UNBOUND_PER_CPU  = 4,      /* 4 * #cpus for unbound wq */
318     WQ_DFL_ACTIVE       = WQ_MAX_ACTIVE / 2,
319 };
```

