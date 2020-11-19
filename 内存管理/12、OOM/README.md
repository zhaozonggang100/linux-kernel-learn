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
		oom_score：oom发生时进程的分数
```

### 2、aarch64  OOM触发过程源码分析(UMA架构)

```c
alloc_pages(gfp_mask, order)
 |-> alloc_pages_node
  |-> __alloc_pages_node
    |-> __alloc_pages
     |-> __alloc_pages_nodemask
      |-> __alloc_pages_slowpath	// get_page_from_freelist分配失败进入慢速内存分配
       |-> __alloc_pages_may_oom    // reclaim failed，可能要触发OOM killer进程
    	|-> out_of_memory			// 执行oom
    
```

- 1、__alloc_pages_nodemask是buddy的heart 

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

- 2、慢速分配通道

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

