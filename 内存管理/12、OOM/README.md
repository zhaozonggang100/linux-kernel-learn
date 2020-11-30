参考：
> https://segmentfault.com/a/1190000008268803  
> https://www.cnblogs.com/arnoldlu/p/8567559.html  
> https://www.cnblogs.com/arnoldlu/p/8568330.html#oom    


### １、文件系统配置  
```
1、/proc/sys/vm/panic_on_oom
		2：申请不到内存一定会出发panic
		1：有可能触发panic
		0：继续检查/proc/sys/vm/oom_kill_allocating_task文件
		
2、/proc/sys/vm/oom_kill_allocating_task
		1：kill掉当前申请内存的进程 
		0：内核将检查每个进程的分数，分数最高（占用内存最大的）的进程将被kill掉
		
3、当被kill掉之后如果/proc/sys/vm/oom_dump_tasks为1且系统的rlimit中设置了core文件大小，将会由/proc/sys/kernel/core_pattern里面指定的程序生成core dump文件，这个文件里将包含pid, uid, tgid, vm size, rss, nr_ptes, nr_pmds, swapents, oom_score_adj、score, name等内容，拿到这个core文件之后，可以做一些分析，看为什么这个进程被选中kill掉。

4、/proc/sys/kernel/panic
		发生panic之后系统重启的时间，单位秒
		
5、/proc/<pid>/
		oom_adj：oom_score中的值会根据这个值来调整（-17->15）
		oom_score：oom发生时进程的分数，经过oom_badness()计算得出对当前进程消耗页面数目，然后相对于totalpages归一化到1000
		oom_score_adj：可读写，范围是-1000~1000。内核中用于计算进程badness points参数。
		
6、/proc/sys/vm/overcommit_memory
	overcommit_memory ==0，系统默认设置，释放较少物理内存，使得oom-kill机制运作比较明显。
	Heuristic overcommit handling. 这是缺省值，它允许overcommit，但过于明目张胆的overcommit会被拒绝，比如malloc一次性申请的内存大小就超过了系统总内存。
	Heuristic的意思是“试探式的”，内核利用某种算法猜测你的内存申请是否合理，它认为不合理就会拒绝overcommit。

 	overcommit_memory == 1，会从buffer中释放较多物理内存，oom-kill也会继续起作用；允许overcommit，对内存申请来者不拒。

 	overcommit_memory == 2，物理内存使用完后，打开任意一个程序均显示内存不足；
	禁止overcommit。CommitLimit 就是overcommit的阈值，申请的内存总数超过CommitLimit的话就算是overcommit。
	也就是说，如果overcommit_memory==2时，内存耗尽时，oom-kill是不会起作用的，系统不会再打开其他程序了，只有等待正在运行的进程释放内存。
	
7、/sys/kernel/debug/tracing/events/oom/

8、/proc/sys/vm/min_free_kbytes
	最小空闲字节数，转换为page（/4096）就是最低水位线
	https://blog.csdn.net/petib_wangwei/article/details/75135686
	
9、/proc/sys/vm/drop_caches
	页缓存清除
	
10、/proc/sys/vm/oom_dump_tasks
	如果设置，则dump_tasks()显示当前系统所有进程内存使用状态
	
11、/proc/sys/vm/oom_kill_allocating_task
	如果设置了sysctl_oom_kill_allocating_task，在当前进程满足一定条件下，优先选择当前进程作为OOM杀死对象。
	
12、/proc/sys/vm/panic_on_oom
	oom触发时是否直接panic而不去杀进程
	
13、/proc/sys/vm/admin_reserve_kbytes
	给有cap_sys_admin权限的用户保留的内存数量，默认值是min(free pages * 3%, 8MB)。这些内存是为了给管理员登录和杀死进程恢复系统提供足够的内存.

14、/proc/sys/vm/dirty_background_ratio
	当脏页的比例（free+可回收内存）超过指定值时，启用kdmflush线程
	
15、/proc/sys/vm/oom_kill_allocating_task
	发生OOM时是否杀死触发OOM的线程
	
16、/proc/zoneinfo
	其中的min|low|high是page单位
	
17、/proc/sys/vm/lowmem_reserve_ratio
	zone中各个区域相对于高层zone保留的page比例
	默认值：256 32（只是UMA只有NORAML|DMA的结果）
	
18、/proc/apgetypeinfo
	查看zone中各种各种可移动类型的free pages数量（zone->free_area->free_list[MIGRAT_TYPE]）
```

### 2、OOM触发条件

```
1、VM里面分配不出更多的page
2、用户地址空间不足
```

### 3、OOM的坏处

```
1、可能进程由于等待IO资源，而没有释放内存，直接将进程杀掉
2、系统正在将一些页面swap到交换分区，可能需要一些时间，结果直接杀进程
```

### 4、aarch64  OOM触发过程源码分析(UMA架构)

```c
1、情况1，正常分配页面，页面不足造成OOM
alloc_pages(gfp_mask, order)
 |-> alloc_pages_node
  |-> __alloc_pages_node
    |-> __alloc_pages
     |-> __alloc_pages_nodemask
      |-> __alloc_pages_slowpath	// get_page_from_freelist分配失败进入慢速内存分配
       |-> __alloc_pages_may_oom    // reclaim failed，可能要触发OOM killer进程
    	|-> out_of_memory			// 执行oom
    	 |->select_bad_process		// 选择最'bad'进程
          |->oom_scan_process_thread
           |->oom_badness			// 计算当前进程有多'badness'
            |->oom_kill_process		// 杀死选中的进程
    
 2、情况2，run out of memory 产生VM_FAULT_OOM错误，就进入pagefault_out_of_memory
    el0_sync-->el0_ia				// 通过软中断进入EL0异常处理函数接着调用el0_ia（EL0指
    										// 令异常） /arch/arm64/kernel/entry.S		
    |-->do_el0_ia_bp_hardening			
     |-->do_mem_abort 						// do_mem_abort是在arm体系架构中，当出现缺页异常，或											   // 者访问空指针后arm的中断异常处理，设置好进程的信号集
      |--> do_translation_fault				// 出现0/1/2/3级页表转换错误时，会调用		
    										// do_translation_fault，实际中	
    										//do_translation_fault最终也会调用到
    										// do_page_fault；
       |--> do_page_fault					// 
        |--> pagefault_out_of_memory()		// 产生VM_FAULT_OOM错误
         |--> mem_cgroup_oom_synchronize
          |--> out_of_memory
 	https://www.lagou.com/lgeduarticle/87749.html
	https://www.taodudu.cc/news/show-1685780.html
	https://blog.csdn.net/yang1349day/article/details/80227665
	https://blog.csdn.net/zsj100213/article/details/82381151
	https://bugzilla.mozilla.org/show_bug.cgi?id=989572
	https://blog.csdn.net/longwang155069/article/details/104789808
```

- 1.1、__alloc_pages_nodemask是buddy的heart ，一般OOM都是由于分配不到页面而触发的

```c
4358 /*
4359  * This is the 'heart' of the zoned buddy allocator.   mm/page_alloc.c
4360  */
4361 struct page *
4362 __alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order, int preferred_nid,
4363                             nodemask_t *nodemask)
4364 {
4365     struct page *page;
4366     unsigned int alloc_flags = ALLOC_WMARK_LOW;
4367     gfp_t alloc_mask; /* The gfp_t that was actually used for allocation */
4368     struct alloc_context ac = { };
4369 
4370     /*
4371      * There are several places where we assume that the order value is sane
4372      * so bail out early if the request is out of bound.
4373      */
4374     if (unlikely(order >= MAX_ORDER)) {
4375         WARN_ON_ONCE(!(gfp_mask & __GFP_NOWARN));
4376         return NULL;
4377     }
4378 
4379     gfp_mask &= gfp_allowed_mask;
4380     alloc_mask = gfp_mask;
4381     if (!prepare_alloc_pages(gfp_mask, order, preferred_nid, nodemask, &ac, &alloc_mask, &alloc_flags))
4382         return NULL;
4383 
4384     finalise_ac(gfp_mask, order, &ac);
4385 
4386     /* First allocation attempt */ 
    	 // 内存充足时这一步从node的freelist即可分配出指定order的page
4387     page = get_page_from_freelist(alloc_mask, order, alloc_flags, &ac);
4388     if (likely(page))
4389         goto out;
4390 
4391     /*
4392      * Apply scoped allocation constraints. This is mainly about GFP_NOFS
4393      * resp. GFP_NOIO which has to be inherited for all allocation requests
4394      * from a particular context which has been marked by
4395      * memalloc_no{fs,io}_{save,restore}.
4396      */
4397     alloc_mask = current_gfp_context(gfp_mask);
4398     ac.spread_dirty_pages = false;
4399 
4400     /*
4401      * Restore the original nodemask if it was potentially replaced with
4402      * &cpuset_current_mems_allowed to optimize the fast-path attempt.
4403      */
4404     if (unlikely(ac.nodemask != nodemask))
4405         ac.nodemask = nodemask;
4406 
    	 // 分配失败，进入慢速分配通道
4407     page = __alloc_pages_slowpath(alloc_mask, order, &ac);
4408 
4409 out:
4410     if (memcg_kmem_enabled() && (gfp_mask & __GFP_ACCOUNT) && page &&
4411         unlikely(memcg_kmem_charge(page, gfp_mask, order) != 0)) {
4412         __free_pages(page, order);
4413         page = NULL;
4414     }
4415 
4416     trace_mm_page_alloc(page, order, alloc_mask, ac.migratetype);
4417 
4418     return page;
4419 }
4420 EXPORT_SYMBOL(__alloc_pages_nodemask);
```

- 1.2、慢速分配通道

```c
4056 static inline struct page *
4057 __alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
4058                         struct alloc_context *ac)
4059 {
4060     bool can_direct_reclaim = gfp_mask & __GFP_DIRECT_RECLAIM;
4061     const bool costly_order = order > PAGE_ALLOC_COSTLY_ORDER;
4062     struct page *page = NULL;
4063     unsigned int alloc_flags;
4064     unsigned long did_some_progress;
4065     enum compact_priority compact_priority;
4066     enum compact_result compact_result;
4067     int compaction_retries;
4068     int no_progress_loops;
4069     unsigned int cpuset_mems_cookie;
4070     int reserve_flags;
4071 
4072     /*
4073      * We also sanity check to catch abuse of atomic reserves being used by
4074      * callers that are not in atomic context.
4075      */
4076     if (WARN_ON_ONCE((gfp_mask & (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)) ==
4077                 (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)))
4078         gfp_mask &= ~__GFP_ATOMIC;
4079 
4080 retry_cpuset:
4081     compaction_retries = 0;
4082     no_progress_loops = 0;
4083     compact_priority = DEF_COMPACT_PRIORITY;
4084     cpuset_mems_cookie = read_mems_allowed_begin();
4085 
4086     /*
4087      * The fast path uses conservative alloc_flags to succeed only until
4088      * kswapd needs to be woken up, and to avoid the cost of setting up
4089      * alloc_flags precisely. So we do that now.
4090      */
4091     alloc_flags = gfp_to_alloc_flags(gfp_mask);
4092 
4093     /*
4094      * We need to recalculate the starting point for the zonelist iterator
4095      * because we might have used different nodemask in the fast path, or
4096      * there was a cpuset modification and we are retrying - otherwise we
4097      * could end up iterating over non-eligible zones endlessly.
4098      */
4099     ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
4100                     ac->high_zoneidx, ac->nodemask);
4101     if (!ac->preferred_zoneref->zone)
4102         goto nopage;
4103 
4104     if (gfp_mask & __GFP_KSWAPD_RECLAIM)
4105         wake_all_kswapds(order, ac);
4106 
4107     /*
4108      * The adjusted alloc_flags might result in immediate success, so try
4109      * that first
4110      */
4111     page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
4112     if (page)
4113         goto got_pg;
4114 
4115     /*
4116      * For costly allocations, try direct compaction first, as it's likely
4117      * that we have enough base pages and don't need to reclaim. For non-
4118      * movable high-order allocations, do that as well, as compaction will
4119      * try prevent permanent fragmentation by migrating from blocks of the
4120      * same migratetype.
4121      * Don't try this for allocations that are allowed to ignore
4122      * watermarks, as the ALLOC_NO_WATERMARKS attempt didn't yet happen.
4123      */
4124     if (can_direct_reclaim &&
4125             (costly_order ||
4126                (order > 0 && ac->migratetype != MIGRATE_MOVABLE))
4127             && !gfp_pfmemalloc_allowed(gfp_mask)) {
4128         page = __alloc_pages_direct_compact(gfp_mask, order,
4129                         alloc_flags, ac,
4130                         INIT_COMPACT_PRIORITY,
4131                         &compact_result);
4132         if (page)
4133             goto got_pg;
4134 
4135         /*
4136          * Checks for costly allocations with __GFP_NORETRY, which
4137          * includes THP page fault allocations
4138          */
4139         if (costly_order && (gfp_mask & __GFP_NORETRY)) {
4140             /*
4141              * If compaction is deferred for high-order allocations,
4142              * it is because sync compaction recently failed. If
4143              * this is the case and the caller requested a THP
4144              * allocation, we do not want to heavily disrupt the
4145              * system, so we fail the allocation instead of entering
4146              * direct reclaim.
4147              */
4148             if (compact_result == COMPACT_DEFERRED)
4149                 goto nopage;
4150 
4151             /*
4152              * Looks like reclaim/compaction is worth trying, but
4153              * sync compaction could be very expensive, so keep
4154              * using async compaction.
4155              */
4156             compact_priority = INIT_COMPACT_PRIORITY;
4157         }
4158     }
4159 
4160 retry:
4161     /* Ensure kswapd doesn't accidentally go to sleep as long as we loop */
4162     if (gfp_mask & __GFP_KSWAPD_RECLAIM)
4163         wake_all_kswapds(order, ac);
4164 
4165     reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
4166     if (reserve_flags)
4167         alloc_flags = reserve_flags;
4168 
4169     /*
4170      * Reset the zonelist iterators if memory policies can be ignored.
4171      * These allocations are high priority and system rather than user
4172      * orientated.
4173      */
4174     if (!(alloc_flags & ALLOC_CPUSET) || reserve_flags) {
4175         ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
4176                     ac->high_zoneidx, ac->nodemask);
4177     }
4178 
4179     /* Attempt with potentially adjusted zonelist and alloc_flags */
4180     page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
4181     if (page)
4182         goto got_pg;
4183 
4184     /* Caller is not willing to reclaim, we can't balance anything */
4185     if (!can_direct_reclaim)
4186         goto nopage;
4187 
4188     /* Avoid recursion of direct reclaim */
4189     if (current->flags & PF_MEMALLOC)
4190         goto nopage;
4191 
4192     if (fatal_signal_pending(current) && !(gfp_mask & __GFP_NOFAIL) &&
4193             (gfp_mask & __GFP_FS))
4194         goto nopage;
4195 
4196     /* Try direct reclaim and then allocating */
4197     page = __alloc_pages_direct_reclaim(gfp_mask, order, alloc_flags, ac,
4198                             &did_some_progress);
4199     if (page)
4200         goto got_pg;
4201 
4202     /* Try direct compaction and then allocating */
4203     page = __alloc_pages_direct_compact(gfp_mask, order, alloc_flags, ac,
4204                     compact_priority, &compact_result);
4205     if (page)
4206         goto got_pg;
4207 
4208     /* Do not loop if specifically requested */
4209     if (gfp_mask & __GFP_NORETRY)
4210         goto nopage;
4211 
4212     /*
4213      * Do not retry costly high order allocations unless they are
4214      * __GFP_RETRY_MAYFAIL
4215      */
4216     if (costly_order && !(gfp_mask & __GFP_RETRY_MAYFAIL))
4217         goto nopage;
4218 
4219     if (should_reclaim_retry(gfp_mask, order, ac, alloc_flags,
4220                  did_some_progress > 0, &no_progress_loops))
4221         goto retry;
4222 
4223     /*
4224      * It doesn't make any sense to retry for the compaction if the order-0
4225      * reclaim is not able to make any progress because the current
4226      * implementation of the compaction depends on the sufficient amount
4227      * of free memory (see __compaction_suitable)
4228      */
4229     if ((did_some_progress > 0 || lmk_kill_possible()) &&
4230             should_compact_retry(ac, order, alloc_flags,
4231                 compact_result, &compact_priority,
4232                 &compaction_retries))
4233         goto retry;
4234 
4235 
4236     /* Deal with possible cpuset update races before we start OOM killing */
4237     if (check_retry_cpuset(cpuset_mems_cookie, ac))
4238         goto retry_cpuset;
4239 
4240     /* Reclaim has failed us, start killing things */
4241     page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
4242     if (page)
4243         goto got_pg;
4244 
4245     /* Avoid allocations with no watermarks from looping endlessly */
4246     if (tsk_is_oom_victim(current) &&
4247         (alloc_flags == ALLOC_OOM ||
4248          (gfp_mask & __GFP_NOMEMALLOC)))
4249         goto nopage;
4250 
4251     /* Retry as long as the OOM killer is making progress */
4252     if (did_some_progress) {
4253         no_progress_loops = 0;
4254         goto retry;
4255     }
4256 
4257 nopage:
4258     /* Deal with possible cpuset update races before we fail */
4259     if (check_retry_cpuset(cpuset_mems_cookie, ac))
4260         goto retry_cpuset;
4261 
4262     /*
4263      * Make sure that __GFP_NOFAIL request doesn't leak out and make sure
4264      * we always retry
4265      */
4266     if (gfp_mask & __GFP_NOFAIL) {
4267         /*
4268          * All existing users of the __GFP_NOFAIL are blockable, so warn
4269          * of any new users that actually require GFP_NOWAIT
4270          */
4271         if (WARN_ON_ONCE(!can_direct_reclaim))
4272             goto fail;
4273 
4274         /*
4275          * PF_MEMALLOC request from this context is rather bizarre
4276          * because we cannot reclaim anything and only can loop waiting
4277          * for somebody to do a work for us
4278          */
4279         WARN_ON_ONCE(current->flags & PF_MEMALLOC);
4280 
4281         /*
4282          * non failing costly orders are a hard requirement which we
4283          * are not prepared for much so let's warn about these users
4284          * so that we can identify them and convert them to something
4285          * else.
4286          */
4287         WARN_ON_ONCE(order > PAGE_ALLOC_COSTLY_ORDER);
4288 
4289         /*
4290          * Help non-failing allocations by giving them access to memory
4291          * reserves but do not use ALLOC_NO_WATERMARKS because this
4292          * could deplete whole memory reserves which would just make
4293          * the situation worse
4294          */
4295         page = __alloc_pages_cpuset_fallback(gfp_mask, order, ALLOC_HARDER, ac);
4296         if (page)
4297             goto got_pg;
4298 
4299         cond_resched();
4300         goto retry;
4301     }
4302 fail:
4303     warn_alloc(gfp_mask, ac->nodemask,
4304             "page allocation failure: order:%u", order);
4305 got_pg:
4306     return page;
4307 }
```

- 1.3、oom入口__alloc_pages_may_oom

```c
2835 static inline struct page *
2836 __alloc_pages_may_oom(gfp_t gfp_mask, unsigned int order,
2837     const struct alloc_context *ac, unsigned long *did_some_progress)
2838 {
2839     struct oom_control oc = {		// oom控制结构
2840         .zonelist = ac->zonelist,
2841         .nodemask = ac->nodemask,
2842         .gfp_mask = gfp_mask,
2843         .order = order,
2844     };
2845     struct page *page;
2846 
2847     *did_some_progress = 0;
2848 
2849     /*
2850      * Acquire the oom lock.  If that fails, somebody else is
2851      * making progress for us.
2852      */
2853     if (!mutex_trylock(&oom_lock)) {
2854         *did_some_progress = 1;
2855         schedule_timeout_uninterruptible(1);
2856         return NULL;
2857     }
2858 
2859     /*
2860      * Go through the zonelist yet one more time, keep very high watermark
2861      * here, this is only to catch a parallel oom killing, we must fail if
2862      * we're still under heavy pressure.
2863      */
    	 // 再次使用高水位检查一次，是否需要启动OOM流程。
2864     page = get_page_from_freelist(gfp_mask | __GFP_HARDWALL, order,
2865                     ALLOC_WMARK_HIGH|ALLOC_CPUSET, ac);
2866     if (page)
2867         goto out;
2868 	
         // __GFP_NOFAIL是不允许内存申请失败的情况，下面都是允许失败的处理。
2869     if (!(gfp_mask & __GFP_NOFAIL)) {
2870         /* Coredumps can quickly deplete all memory reserves */
2871         if (current->flags & PF_DUMPCORE)
2872             goto out;
2873         /* The OOM killer will not help higher order allocs */
    		 // order超过3的申请失败，不会启动OOM回收。
2874         if (order > PAGE_ALLOC_COSTLY_ORDER)
2875             goto out;
2876         /* The OOM killer does not needlessly kill tasks for lowmem */
2877         if (ac->high_zoneidx < ZONE_NORMAL)
2878             goto out;
2879         /* The OOM killer does not compensate for IO-less reclaim */
2880         if (!(gfp_mask & __GFP_FS)) {
2881             /*
2882              * XXX: Page reclaim didn't yield anything,
2883              * and the OOM killer can't be invoked, but
2884              * keep looping as per tradition.
2885              */
2886             *did_some_progress = 1;
2887             goto out;
2888         }
2889         if (pm_suspended_storage())
2890             goto out;
2891         /* The OOM killer may not free memory on a specific node */
2892         if (gfp_mask & __GFP_THISNODE)
2893             goto out;
2894     }
2895     /* Exhausted what can be done so it's blamo time */
    	 // 经过上面各种情况，任然需要进行OOM处理。调用out_of_memory()。
2896     if (out_of_memory(&oc) || WARN_ON_ONCE(gfp_mask & __GFP_NOFAIL))
2897         *did_some_progress = 1;
2898 out:
2899     mutex_unlock(&oom_lock);
2900     return page;
2901 }
```

- 1.4、开始执行out_of_memory收割、回收内存或者杀进程，对进程打分以及杀死最高评分进程

> ```c
> 663 /**
> 664  * out_of_memory - kill the "best" process when we run out of memory
> 665  * @oc: pointer to struct oom_control
> 666  *
> 667  * If we run out of memory, we have the choice between either
> 668  * killing a random task (bad), letting the system crash (worse)
> 669  * OR try to be smart about which process to kill. Note that we
> 670  * don't have to be perfect here, we just have to be good.
> 671  */
> 672 bool out_of_memory(struct oom_control *oc)
> 673 {
> 674     struct task_struct *p;
> 675     unsigned long totalpages;
> 676     unsigned long freed = 0;
> 677     unsigned int uninitialized_var(points);
> 678     enum oom_constraint constraint = CONSTRAINT_NONE;
> 679 
>     	// 在freeze_processes会将其置位，即禁止OOM；在thaw_processes会将其清零，即打开OOM。所以，如果在冻结过程，不允许OOM。
> 680     if (oom_killer_disabled)
> 681         return false;
> 682 
> 683     blocking_notifier_call_chain(&oom_notify_list, 0, &freed);
> 684     if (freed > 0)
> 685         /* Got some memory back in the last second. */
> 686         return true;
> 687 
> 688     /*
> 689      * If current has a pending SIGKILL or is exiting, then automatically
> 690      * select it.  The goal is to allow it to allocate so that it may
> 691      * quickly exit and free its memory.
> 692      *
> 693      * But don't select if current has already released its mm and cleared
> 694      * TIF_MEMDIE flag at exit_mm(), otherwise an OOM livelock may occur.
> 695      */ 
> 696     if (current->mm &&
> 697         (fatal_signal_pending(current) || task_will_free_mem(current))) {
> 698         mark_oom_victim(current);
> 699         return true;
> 700     }
> 701 
> 702     /*
> 703      * Check if there were limitations on the allocation (only relevant for
> 704      * NUMA) that may require different handling.
> 705      */
>     	// 未定义CONFIG_NUMA返回CONSTRAINT_NONE
> 706     constraint = constrained_alloc(oc, &totalpages);
> 707     if (constraint != CONSTRAINT_MEMORY_POLICY)
> 708         oc->nodemask = NULL;
>     
>     	// 检查sysctl_panic_on_oom设置，以及是否由sysrq触发，来决定是否触发panic。
> 709     check_panic_on_oom(oc, constraint, NULL);
> 710 
>     	// 如果设置了sysctl_oom_kill_allocating_task，那么当内存耗尽时，会把当前申请内存分配的进程杀掉
> 711     if (sysctl_oom_kill_allocating_task && current->mm &&
> 712         !oom_unkillable_task(current, NULL, oc->nodemask) &&
> 713         current->signal->oom_score_adj != OOM_SCORE_ADJ_MIN) {
> 714         get_task_struct(current);
> 715         oom_kill_process(oc, current, 0, totalpages, NULL,
> 716                  "Out of memory (oom_kill_allocating_task)");
> 717         return true;
> 718     }
> 719 
>     	// ******遍历所有进程，进程下的线程，查找合适的候选进程。即得分最高的候选进程。***
> 720     p = select_bad_process(oc, &points, totalpages);
> 721     /* Found nothing?!?! Either we hang forever, or we panic. */
>     	// 如果没有合适候选进程，并且OOM不是由sysrq触发的，进入panic。
> 722     if (!p && !is_sysrq_oom(oc)) {
> 723         dump_header(oc, NULL, NULL);
> 724         panic("Out of memory and no killable processes...\n");
> 725     }
> 726     if (p && p != (void *)-1UL) {
> 727         oom_kill_process(oc, p, points, totalpages, NULL,
> 728                  "Out of memory");
> 729         /*
> 730          * Give the killed process a good chance to exit before trying
> 731          * to allocate memory again.
> 732          */
> 733         schedule_timeout_killable(1);
> 734     }
> 735     return true;
> 736 }
> 
> 296 /*
> 297  * Simple selection loop. We chose the process with the highest
> 298  * number of 'points'.  Returns -1 on scan abort.
> 299  */
> 300 static struct task_struct *select_bad_process(struct oom_control *oc,
> 301         unsigned int *ppoints, unsigned long totalpages)
> 302 {
> 303     struct task_struct *g, *p;
> 304     struct task_struct *chosen = NULL;
> 305     unsigned long chosen_points = 0;
> 306     
> 307     rcu_read_lock();
> 308     for_each_process_thread(g, p) {
> 309         unsigned int points;
> 310 
> 311         switch (oom_scan_process_thread(oc, p, totalpages)) {
> 312         case OOM_SCAN_SELECT:
> 313             chosen = p;
> 314             chosen_points = ULONG_MAX;
> 315             /* fall through */
> 316         case OOM_SCAN_CONTINUE:
> 317             continue;
> 318         case OOM_SCAN_ABORT:
> 319             rcu_read_unlock();
> 320             return (struct task_struct *)(-1UL);
> 321         case OOM_SCAN_OK:
> 322             break;
> 323         };
> 324         points = oom_badness(p, NULL, oc->nodemask, totalpages);
> 325         if (!points || points < chosen_points)
> 326             continue;
> 327         /* Prefer thread group leaders for display purposes */
> 328         if (points == chosen_points && thread_group_leader(chosen))
> 329             continue;
> 330 
> 331         chosen = p;
> 332         chosen_points = points;
> 333     }
> 334     if (chosen)
> 335         get_task_struct(chosen);
> 336     rcu_read_unlock();
> 337 
> 338     *ppoints = chosen_points * 1000 / totalpages;
> 339     return chosen;
> 340 }
> 
> 503 #define K(x) ((x) << (PAGE_SHIFT-10))
> 504 /*          
> 505  * Must be called while holding a reference to p, which will be released upon
> 506  * returning.
> 507  */         
> 508 void oom_kill_process(struct oom_control *oc, struct task_struct *p,
> 509               unsigned int points, unsigned long totalpages,
> 510               struct mem_cgroup *memcg, const char *message)
> 511 {       
> 512     struct task_struct *victim = p;
> 513     struct task_struct *child;
> 514     struct task_struct *t;
> 515     struct mm_struct *mm;
> 516     unsigned int victim_points = 0;
> 517     static DEFINE_RATELIMIT_STATE(oom_rs, DEFAULT_RATELIMIT_INTERVAL,
> 518                           DEFAULT_RATELIMIT_BURST);
> 519 
> 520     /*
> 521      * If the task is already exiting, don't alarm the sysadmin or kill
> 522      * its children or threads, just set TIF_MEMDIE so it can die quickly
> 523      */
> 524     task_lock(p);
>     	// 对于非coredump正处于退出状态的线程，标注TIF_MEMDIE并唤醒reaper线程进行收割，然后退出。
> 525     if (p->mm && task_will_free_mem(p)) {
> 526         mark_oom_victim(p);
> 527         task_unlock(p);
> 528         put_task_struct(p);
> 529         return;
> 530     }
> 531     task_unlock(p);
> 532 
>     	// 在kill进程之前，将系统栈信息、内存信息、所有进程的内存消耗情况打印。
>     	// 这就是经常看到的call trace
> 533     if (__ratelimit(&oom_rs))
> 534         dump_header(oc, p, memcg);
> 535 
>     	// 输出将要kill掉的进程名、pid、score。
> 536     pr_err("%s: Kill process %d (%s) score %u or sacrifice child\n",
> 537         message, task_pid_nr(p), p->comm, points);
> 538 
> 539     /*
> 540      * If any of p's children has a different mm and is eligible for kill,
> 541      * the one with the highest oom_badness() score is sacrificed for its
> 542      * parent.  This attempts to lose the minimal amount of work done while
> 543      * still freeing memory.
> 544      */
> 545     read_lock(&tasklist_lock);
> 546     for_each_thread(p, t) {
> 547         list_for_each_entry(child, &t->children, sibling) {
> 548             unsigned int child_points;
> 549 
> 550             if (process_shares_mm(child, p->mm))
> 551                 continue;
> 552             /*
> 553              * oom_badness() returns 0 if the thread is unkillable
> 554              */
> 555             child_points = oom_badness(child, memcg, oc->nodemask,
> 556                                 totalpages);
> 557             if (child_points > victim_points) {
> 558                 put_task_struct(victim);
> 559                 victim = child;
> 560                 victim_points = child_points;
> 561                 get_task_struct(victim);
> 562             }
> 563         }
> 564     }
> 565     read_unlock(&tasklist_lock);
> 566 
> 567     p = find_lock_task_mm(victim);
> 568     if (!p) {
> 569         put_task_struct(victim);
> 570         return;
> 571     } else if (victim != p) {
> 572         get_task_struct(p);
> 573         put_task_struct(victim);
> 574         victim = p;
> 575     }
> 576 
> 577     /* Get a reference to safely compare mm after task_unlock(victim) */
> 578     mm = victim->mm;
> 579     atomic_inc(&mm->mm_count);
> 580     /*
> 581      * We should send SIGKILL before setting TIF_MEMDIE in order to prevent
> 582      * the OOM victim from depleting the memory reserves from the user
> 583      * space under its control.
> 584      */
> 585     do_send_sig_info(SIGKILL, SEND_SIG_FORCED, victim, true);
> 586     mark_oom_victim(victim);
> 587     pr_err("Killed process %d (%s) total-vm:%lukB, anon-rss:%lukB, file-rss:%lukB\n",
> 588         task_pid_nr(victim), victim->comm, K(victim->mm->total_vm),
> 589         K(get_mm_counter(victim->mm, MM_ANONPAGES)),
> 590         K(get_mm_counter(victim->mm, MM_FILEPAGES)));
> 591     task_unlock(victim);
> 592 
> 593     /*
> 594      * Kill all user processes sharing victim->mm in other thread groups, if
> 595      * any.  They don't get access to memory reserves, though, to avoid
> 596      * depletion of all memory.  This prevents mm->mmap_sem livelock when an
> 597      * oom killed thread cannot exit because it requires the semaphore and
> 598      * its contended by another thread trying to allocate memory itself.
> 599      * That thread will now get access to memory reserves since it has a
> 600      * pending fatal signal.
> 601      */
> 602     rcu_read_lock();
> 603     for_each_process(p) {
> 604         if (!process_shares_mm(p, mm))
> 605             continue;
> 606         if (same_thread_group(p, victim))
> 607             continue;
> 608         if (unlikely(p->flags & PF_KTHREAD))
> 609             continue;
> 610         if (is_global_init(p))
> 611             continue;
> 612         if (p->signal->oom_score_adj == OOM_SCORE_ADJ_MIN)
> 613             continue;
> 614 
> 615         do_send_sig_info(SIGKILL, SEND_SIG_FORCED, p, true);
> 616     }
> 617     rcu_read_unlock();
> 618 
> 619     mmdrop(mm);
> 620     put_task_struct(victim);
> 621 }
> 622 #undef K
>     
> 624 /*
> 625  * Determines whether the kernel must panic because of the panic_on_oom sysctl.
> 	 * 如果设置了panic_on_oom，则在发生OOM时直接panic，而不会去杀进程
> 626  */
> 627 void check_panic_on_oom(struct oom_control *oc, enum oom_constraint constraint,
> 628             struct mem_cgroup *memcg)
> 629 {
> 630     if (likely(!sysctl_panic_on_oom))
> 631         return;
> 632     if (sysctl_panic_on_oom != 2) {
> 633         /*
> 634          * panic_on_oom == 1 only affects CONSTRAINT_NONE, the kernel
> 635          * does not panic for cpuset, mempolicy, or memcg allocation
> 636          * failures.
> 637          */
> 638         if (constraint != CONSTRAINT_NONE)
> 639             return;
> 640     }
> 641     /* Do not panic for oom kills triggered by sysrq */
>     	// 确定不是因为sysrq触发的panic
> 642     if (is_sysrq_oom(oc))
> 643         return;
>     	// 打印调用栈
> 644     dump_header(oc, NULL, memcg);
> 645     panic("Out of memory: %s panic_on_oom is enabled\n",
> 646         sysctl_panic_on_oom == 2 ? "compulsory" : "system-wide");
> 647 }
> ```

- 1.5、dump_header输出call trace

```c
386 static void dump_header(struct oom_control *oc, struct task_struct *p,
387             struct mem_cgroup *memcg)
388 {   
389     pr_warning("%s invoked oom-killer: gfp_mask=0x%x, order=%d, oom_score_adj=%hd\n",
390         current->comm, oc->gfp_mask, oc->order,
391         current->signal->oom_score_adj);
392     cpuset_print_current_mems_allowed();
    	// // 输出当前现场的栈信息。
393     dump_stack();	
394     if (memcg)
395         mem_cgroup_print_oom_info(memcg, p);
396     else
    		// 输出整个系统的内存使用情况。
397         show_mem(SHOW_MEM_FILTER_NODES);
398     if (sysctl_oom_dump_tasks)
    		// 显示系统所有进程的内存使用情况。
399         dump_tasks(memcg, oc->nodemask);
400 }
```



- 2.1、产生缺页中断

```c
299 static int __kprobes do_page_fault(unsigned long addr, unsigned int esr,
300                    struct pt_regs *regs)
301 {
302     struct task_struct *tsk;
303     struct mm_struct *mm;
304     int fault, sig, code;
305     unsigned long vm_flags = VM_READ | VM_WRITE | VM_EXEC;
306     unsigned int mm_flags = FAULT_FLAG_ALLOW_RETRY | FAULT_FLAG_KILLABLE;
307 
308     if (notify_page_fault(regs, esr))
309         return 0;
310 
311     tsk = current;
312     mm  = tsk->mm;
313 
314     /* Enable interrupts if they were enabled in the parent context. */
315     if (interrupts_enabled(regs))
316         local_irq_enable();
317 
318     /*
319      * If we're in an interrupt or have no user context, we must not take
320      * the fault.
321      */
322     if (faulthandler_disabled() || !mm)
323         goto no_context;
324 
325     if (user_mode(regs))
326         mm_flags |= FAULT_FLAG_USER;
327 
328     if (is_el0_instruction_abort(esr)) {
329         vm_flags = VM_EXEC;
330     } else if (((esr & ESR_ELx_WNR) && !(esr & ESR_ELx_CM)) ||
331             ((esr & ESR_ELx_CM) && !(mm_flags & FAULT_FLAG_USER))) {
332         vm_flags = VM_WRITE;
333         mm_flags |= FAULT_FLAG_WRITE;
334     }
335 
336     if (addr < USER_DS && is_permission_fault(esr, regs)) {
337         if (is_el1_instruction_abort(esr))
338             die("Attempting to execute userspace memory", regs, esr);
339 
340         if (!search_exception_tables(regs->pc))
341             die("Accessing user space memory outside uaccess.h routines", regs, esr);
342     }
343 
344     /*
345      * As per x86, we may deadlock here. However, since the kernel only
346      * validly references user space from well defined areas of the code,
347      * we can bug out early if this is from code which shouldn't.
348      */
349     if (!down_read_trylock(&mm->mmap_sem)) {
350         if (!user_mode(regs) && !search_exception_tables(regs->pc))
351             goto no_context;
352 retry:
353         down_read(&mm->mmap_sem);
354     } else {
355         /*
356          * The above down_read_trylock() might have succeeded in which
357          * case, we'll have missed the might_sleep() from down_read().
358          */
359         might_sleep();
360 #ifdef CONFIG_DEBUG_VM
361         if (!user_mode(regs) && !search_exception_tables(regs->pc))
362             goto no_context;
363 #endif
364     }
365 
366     fault = __do_page_fault(mm, addr, mm_flags, vm_flags, tsk);
367 
368     /*
369      * If we need to retry but a fatal signal is pending, handle the
370      * signal first. We do not need to release the mmap_sem because it
371      * would already be released in __lock_page_or_retry in mm/filemap.c.
372      */
373     if ((fault & VM_FAULT_RETRY) && fatal_signal_pending(current)) {
374         if (!user_mode(regs))
375             goto no_context;
376         return 0;
377     }
378 
379     /*
380      * Major/minor page fault accounting is only done on the initial
381      * attempt. If we go through a retry, it is extremely likely that the
382      * page will be found in page cache at that point.
383      */
384 
385     perf_sw_event(PERF_COUNT_SW_PAGE_FAULTS, 1, regs, addr);
386     if (mm_flags & FAULT_FLAG_ALLOW_RETRY) {
387         if (fault & VM_FAULT_MAJOR) {
388             tsk->maj_flt++;
389             perf_sw_event(PERF_COUNT_SW_PAGE_FAULTS_MAJ, 1, regs,
390                       addr);
391         } else {
392             tsk->min_flt++;
393             perf_sw_event(PERF_COUNT_SW_PAGE_FAULTS_MIN, 1, regs,
394                       addr);
395         }
396         if (fault & VM_FAULT_RETRY) {
397             /*
398              * Clear FAULT_FLAG_ALLOW_RETRY to avoid any risk of
399              * starvation.
400              */
401             mm_flags &= ~FAULT_FLAG_ALLOW_RETRY;
402             mm_flags |= FAULT_FLAG_TRIED;
403             goto retry;
404         }
405     }
406 
407     up_read(&mm->mmap_sem);
408 
409     /*
410      * Handle the "normal" case first - VM_FAULT_MAJOR / VM_FAULT_MINOR
411      */
412     if (likely(!(fault & (VM_FAULT_ERROR | VM_FAULT_BADMAP |
413                   VM_FAULT_BADACCESS))))
414         return 0;
415 
416     /*
417      * If we are in kernel mode at this point, we have no context to
418      * handle this fault with.
419      */
420     if (!user_mode(regs))
421         goto no_context;
422 
423     if (fault & VM_FAULT_OOM) {		// 1、触发page fault OOM
424         /*
425          * We ran out of memory, call the OOM killer, and return to
426          * userspace (which will retry the fault, or kill us if we got
427          * oom-killed).
428          */
429         pagefault_out_of_memory();
430         return 0;
431     }
432 
433     if (fault & VM_FAULT_SIGBUS) {
434         /*
435          * We had some memory, but were unable to successfully fix up
436          * this page fault.
437          */
438         sig = SIGBUS;
439         code = BUS_ADRERR;
440     } else {
441         /*
442          * Something tried to access memory that isn't in our memory
443          * map.
444          */
445         sig = SIGSEGV;
446         code = fault == VM_FAULT_BADACCESS ?
447             SEGV_ACCERR : SEGV_MAPERR;
448     }
449 
450     __do_user_fault(tsk, addr, esr, sig, code, regs);
451     return 0;
452 
453 no_context:
454     __do_kernel_fault(mm, addr, esr, regs);
455     return 0;
456 }
```

- 2.1、由于VM_FAULT_OOM进入pagefault_out_of_memory

```c
738 /*
739  * The pagefault handler calls here because it is out of memory, so kill a
740  * memory-hogging task.  If any populated zone has ZONE_OOM_LOCKED set, a
741  * parallel oom killing is already in progress so do nothing.
742  */
743 void pagefault_out_of_memory(void)
744 {
745     struct oom_control oc = {
746         .zonelist = NULL,
747         .nodemask = NULL,
748         .gfp_mask = 0,
749         .order = 0,
750     };
751     
752     if (mem_cgroup_oom_synchronize(true))
753         return;
754 
755     if (!mutex_trylock(&oom_lock))
756         return;
757 
758     if (!out_of_memory(&oc)) {
759         /*
760          * There shouldn't be any user tasks runnable while the
761          * OOM killer is disabled, so the current task has to
762          * be a racing OOM victim for which oom_killer_disable()
763          * is waiting for.
764          */
765         WARN_ON(test_thread_flag(TIF_MEMDIE));
766     }
767 
768     mutex_unlock(&oom_lock);
769 
```

