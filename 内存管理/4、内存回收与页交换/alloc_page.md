### 1、调用关系

- 1、alloc_pages(gfp_t gfp_mask, unsigned int order)

- 2、alloc_pages_current(gfp_mask, order)

- 3、__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
             				struct zonelist *zonelist, nodemask_t *nodemask)

- 4、正真分配free  page，buddy的核心

```c
3236 /*
3237  * This is the 'heart' of the zoned buddy allocator.
3238  */
3239 struct page *
3240 __alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
3241             struct zonelist *zonelist, nodemask_t *nodemask)
3242 {
3243     struct zoneref *preferred_zoneref;
3244     struct page *page = NULL;
3245     unsigned int cpuset_mems_cookie;
    	 // ALLOC_WMARK_LOW:最开始尝试从低水位线分配
3246     int alloc_flags = ALLOC_WMARK_LOW|ALLOC_CPUSET|ALLOC_FAIR;
3247     gfp_t alloc_mask; /* The gfp_t that was actually used for allocation */
3248     struct alloc_context ac = {
3249         .high_zoneidx = gfp_zone(gfp_mask),	// 从MASK获取zone
3250         .nodemask = nodemask,			
3251         .migratetype = gfpflags_to_migratetype(gfp_mask),	// 从mask获取移动类型
3252     };
3253 
3254     gfp_mask &= gfp_allowed_mask;
3255 
3256     lockdep_trace_alloc(gfp_mask);
3257 
    	 //如果此次内存分配可以等待（睡眠），那么再深入判断此task是否可以被调度，如果是将主动schedule
3258     might_sleep_if(gfp_mask & __GFP_DIRECT_RECLAIM);
3259 
    	 /*
    	 	未定义CONFIG_FAIL_PAGE_ALLOC，直接返回false，继续向下运行
    	 	为分配失败调试做准备
    	 */
3260     if (should_fail_alloc_page(gfp_mask, order))
3261         return NULL;
3262 
3263     /*
3264      * Check the zones suitable for the gfp_mask contain at least one
3265      * valid zone. It's possible to have an empty zonelist as a result
3266      * of __GFP_THISNODE and a memoryless node
3267      */
3268     if (unlikely(!zonelist->_zonerefs->zone))
3269         return NULL;
3270 
3271     if (IS_ENABLED(CONFIG_CMA) && ac.migratetype == MIGRATE_MOVABLE)
3272         alloc_flags |= ALLOC_CMA;
3273 
3274 retry_cpuset:
    	 // 获取进程顺序锁：current->mems_allowed_seq)
3275     cpuset_mems_cookie = read_mems_allowed_begin();
3276 
3277     /* We set it here, as __alloc_pages_slowpath might have changed it */
    	 // 设置alloc_context表示此次分配的备用zone表
3278     ac.zonelist = zonelist;
3279 
3280     /* Dirty zone balancing only done in the fast path */
3281     ac.spread_dirty_pages = (gfp_mask & __GFP_WRITE);
3282 
3283     /* The preferred zone is used for statistics later */
    	 // 找符合mask分配的zone，找不到直接返回
3284     preferred_zoneref = first_zones_zonelist(ac.zonelist, ac.high_zoneidx,
3285                 ac.nodemask ? : &cpuset_current_mems_allowed,
3286                 &ac.preferred_zone);
3287     if (!ac.preferred_zone)
3288         goto out;
3289     ac.classzone_idx = zonelist_zone_idx(preferred_zoneref);
3290 
    		/*
    			快速分配通道，首先尝试使用低水位线（zone->water[low]）分配
    		*/
3291     /* First allocation attempt */
3292     alloc_mask = gfp_mask|__GFP_HARDWALL;
3293     page = get_page_from_freelist(alloc_mask, order, alloc_flags, &ac);
3294     if (unlikely(!page)) {
3295         /*
3296          * Runtime PM, block IO and its error handling path
3297          * can deadlock because I/O on the device might not
3298          * complete.
3299          */
3300         alloc_mask = memalloc_noio_flags(gfp_mask);
3301         ac.spread_dirty_pages = false;
3302 		 
    		 /*
    		 	使用zone->water[low]分配失败，尝试使用慢速分配通道
    		 */
3303         page = __alloc_pages_slowpath(alloc_mask, order, &ac);
3304     }
3305 
3306     if (kmemcheck_enabled && page)
3307         kmemcheck_pagealloc_alloc(page, order, gfp_mask);
3308 
3309     trace_mm_page_alloc(page, order, alloc_mask, ac.migratetype);
3310 
3311 out:
3312     /*
3313      * When updating a task's mems_allowed, it is possible to race with
3314      * parallel threads in such a way that an allocation can fail while
3315      * the mask is being updated. If a page allocation is about to fail,
3316      * check if the cpuset changed during allocation and if so, retry.
3317      */
3318     if (unlikely(!page && read_mems_allowed_retry(cpuset_mems_cookie)))
3319         goto retry_cpuset;
3320 
3321     return page;
3322 }
3323 EXPORT_SYMBOL(__alloc_pages_nodemask);
```

- 5、快速分配通道，使用低水位线

```c
2628 /*
2629  * get_page_from_freelist goes through the zonelist trying to allocate
2630  * a page.
2631  */    
2632 static struct page *
2633 get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
2634                         const struct alloc_context *ac)
2635 {   
2636     struct zonelist *zonelist = ac->zonelist;
2637     struct zoneref *z;
2638     struct page *page = NULL;
2639     struct zone *zone;
2640     int nr_fair_skipped = 0;
2641     bool zonelist_rescan;
2642     
2643 zonelist_scan:
2644     zonelist_rescan = false;
2645     
2646     /*
2647      * Scan zonelist, looking for a zone with enough free.
2648      * See also __cpuset_node_allowed() comment in kernel/cpuset.c.
2649      */
2650     for_each_zone_zonelist_nodemask(zone, z, zonelist, ac->high_zoneidx,
2651                                 ac->nodemask) {
2652         unsigned long mark;
2653 
    		 // 限制进程只能运行在指定CPU上和在指定的NODE上分配内存
2654         if (cpusets_enabled() &&
2655             (alloc_flags & ALLOC_CPUSET) &&
2656             !cpuset_zone_allowed(zone, gfp_mask))
2657                 continue;
2658         /*
2659          * Distribute pages in proportion to the individual
2660          * zone size to ensure fair page aging.  The zone a
2661          * page was allocated in should have no effect on the
2662          * time the page has in memory before being reclaimed.
2663          */
2664         if (alloc_flags & ALLOC_FAIR) {
2665             if (!zone_local(ac->preferred_zone, zone))
2666                 break;
2667             if (test_bit(ZONE_FAIR_DEPLETED, &zone->flags)) {
2668                 nr_fair_skipped++;
2669                 continue;
2670             }
2671         }
2672         /*
2673          * When allocating a page cache page for writing, we
2674          * want to get it from a zone that is within its dirty
2675          * limit, such that no single zone holds more than its
2676          * proportional share of globally allowed dirty pages.
2677          * The dirty limits take into account the zone's
2678          * lowmem reserves and high watermark so that kswapd
2679          * should be able to balance it without having to
2680          * write pages from its LRU list.
2681          *
2682          * This may look like it could increase pressure on
2683          * lower zones by failing allocations in higher zones
2684          * before they are full.  But the pages that do spill
2685          * over are limited as the lower zones are protected
2686          * by this very same mechanism.  It should not become
2687          * a practical burden to them.
2688          *
2689          * XXX: For now, allow allocations to potentially
2690          * exceed the per-zone dirty limit in the slowpath
2691          * (spread_dirty_pages unset) before going into reclaim,
2692          * which is important when on a NUMA setup the allowed
2693          * zones are together not big enough to reach the
2694          * global limit.  The proper fix for these situations
2695          * will require awareness of zones in the
2696          * dirty-throttling and the flusher threads.
2697          */
    		 // 如果调用者设置了__GFP_WRITE，表示文件系统申请分配一个页缓存用于写文件，那么检查节点的脏页数量是否超过限制，如果超过，不能从这个区域分配页
2698         if (ac->spread_dirty_pages && !zone_dirty_ok(zone))
2699             continue;
2700 
    		 // 检查水位线是否符合要求，如果zone的空闲页数-申请的页数小于低水位线，进行如下处理
2701         mark = zone->watermark[alloc_flags & ALLOC_WMARK_MASK];
2702         if (!zone_watermark_ok(zone, order, mark,
2703                        ac->classzone_idx, alloc_flags)) {
2704             int ret;
2705 
2706             /* Checked here to keep the fast path fast */
2707             BUILD_BUG_ON(ALLOC_NO_WATERMARKS < NR_WMARK);
    			 // 调用者要求不检查水位线，可以从这个zone分配
2708             if (alloc_flags & ALLOC_NO_WATERMARKS)
2709                 goto try_this_zone;
2710 
    			 // 如果没有开启区域回收功能或者当前节点和首选节点之间的距离大于回收距离，那么不能从这个zone分配，//如果本zone不允许回收，更新zone list cache为full，为下一次check节省时间
2711             if (zone_reclaim_mode == 0 ||
2712                 !zone_allows_reclaim(ac->preferred_zone, zone))
2713                 continue;
2714 			
    			 // 从节点回收没有映射到进程虚拟地址空间的文件页和块分配器申请的页，重新监测水位线，如果还是小于水位线，那么不能从这个节点分配页
    			 // 启动回收机制，如果不可以wait，返回ZONE_RECLAIM_NOSCAN，
				 // 否则在local zone或没有关联到其他processor的zone，调用__zone_reclaim进行回收
2715             ret = zone_reclaim(zone, gfp_mask, order);
2716             switch (ret) {
2717             case ZONE_RECLAIM_NOSCAN:
2718                 /* did not scan */
2719                 continue;
2720             case ZONE_RECLAIM_FULL:	//没有分配空间
2721                 /* scanned but unreclaimable */
2722                 continue;
2723             default:
2724                 /* did we reclaim enough */ //成功进行了回收，check水线是否满足要求
2725                 if (zone_watermark_ok(zone, order, mark,
2726                         ac->classzone_idx, alloc_flags))
2727                     goto try_this_zone;
2728 
2729                 continue;
2730             }
2731         }
2732 
    		 // //各种情况check完毕，在本zone进行真正的内存分配动作
2733 try_this_zone:
2734         page = buffered_rmqueue(ac->preferred_zone, zone, order,
2735                 gfp_mask, alloc_flags, ac->migratetype);
    		 // 如果分配成功，初始化页
2736         if (page) {
2737             if (prep_new_page(page, order, gfp_mask, alloc_flags))
2738                 goto try_this_zone;
2739 
2740             /*
2741              * If this is a high-order atomic allocation then check
2742              * if the pageblock should be reserved for the future
2743              */
2744             if (unlikely(order && (alloc_flags & ALLOC_HARDER)))
2745                 reserve_highatomic_pageblock(page, zone, order);
2746 
2747             return page;
2748         }
2749     }
2750 
2751     /*
2752      * The first pass makes sure allocations are spread fairly within the
2753      * local node.  However, the local node might have free pages left
2754      * after the fairness batches are exhausted, and remote zones haven't
2755      * even been considered yet.  Try once more without fairness, and
2756      * include remote zones now, before entering the slowpath and waking
2757      * kswapd: prefer spilling to a remote zone over swapping locally.
2758      */
2759     if (alloc_flags & ALLOC_FAIR) {
2760         alloc_flags &= ~ALLOC_FAIR;
2761         if (nr_fair_skipped) {
2762             zonelist_rescan = true;
2763             reset_alloc_batches(ac->preferred_zone);
2764         }
2765         if (nr_online_nodes > 1)
2766             zonelist_rescan = true;
2767     }
2768 
2769     if (zonelist_rescan)
2770         goto zonelist_scan;
2771 
2772     return NULL;
2773 }


2316 /*
2317  * Allocate a page from the given zone. Use pcplists for order-0 allocations.
2318  */
2319 static inline
2320 struct page *buffered_rmqueue(struct zone *preferred_zone,
2321             struct zone *zone, unsigned int order,
2322             gfp_t gfp_flags, int alloc_flags, int migratetype)
2323 {
2324     unsigned long flags;
2325     struct page *page = NULL;
2326     bool cold = ((gfp_flags & __GFP_COLD) != 0);
2327 	 
    	 // 如果是分配一个页，直接从zone的pcp中搜索
2328     if (likely(order == 0)) {
2329         struct per_cpu_pages *pcp;
2330         struct list_head *list = NULL;
2331 
2332         local_irq_save(flags);
2333         pcp = &this_cpu_ptr(zone->pageset)->pcp;	//取得pcp指针
2334 
2335         /* First try to get CMA pages */
2336         if (migratetype == MIGRATE_MOVABLE &&
2337             gfp_flags & __GFP_CMA) {
2338             list = get_populated_pcp_list(zone, 0, pcp,
2339                     get_cma_migrate_type(), cold);	//取得对应的migrate type pcp list
2340         }
2341 
2342         if (list == NULL) {
2343             /*
2344              * Either CMA is not suitable or there are no free CMA
2345              * pages.
2346              */
    			 // 不要求从CMA区域分配，获取zone的pageset集合链表，内部会判断链表是否为空，空的话需要从buddy批量获取页
2347             list = get_populated_pcp_list(zone, 0, pcp,
2348                 migratetype, cold);
2349             if (unlikely(list == NULL) ||
2350                 unlikely(list_empty(list)))	//如果该list为空
2351                 goto failed;
2352         }
2353 
2354         if (cold)	//分配冷页，不被cache的页，比如用于DMA，从链表尾部分配
2355             page = list_entry(list->prev, struct page, lru);
2356         else	//分配热页，被cache的页，提高效率，从链表头部分配
2357             page = list_entry(list->next, struct page, lru);	//热页从链表头开始查找
2358 
2359         list_del(&page->lru);	// 从lru list删除该页，此时分配的页不能回收
2360         pcp->count--;	// pcp->count值代表本pcp有多少页
2361     } else {
2362         if (unlikely(gfp_flags & __GFP_NOFAIL)) {
2363             /*
2364              * __GFP_NOFAIL is not to be used in new code.
2365              *
2366              * All __GFP_NOFAIL callers should be fixed so that they
2367              * properly detect and handle allocation failures.
2368              *
2369              * We most definitely don't want callers attempting to
2370              * allocate greater than order-1 page units with
2371              * __GFP_NOFAIL.
2372              */
2373             WARN_ON_ONCE(order > 1);
2374         }
2375         spin_lock_irqsave(&zone->lock, flags);	//为操作buddy上锁
2376 
    		 /*
    		 	真正从buddy的free_area链表分配内存，分两种情况
					1. __rmqueue_smallest()：如果order上对应分配策略要求的migrate type list有空间，从第一满足的节点上分配内存，并将剩余的部分add到更小的order链表上
					2. __rmqueue_fallback()：如果对应的migrate type list上没有空间，fallback到其他的type list上，释放一定空间到本migratte type list上，fallback有相应的sequence，再进行和__rmqueue_smallest类似的分配、合并动作
    		 */
2377         page = NULL;
2378         if (alloc_flags & ALLOC_HARDER) {
    			 // 从申请order到最大order挨个尝试
2379             page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);
2380             if (page)
2381                 trace_mm_page_alloc_zone_locked(page, order, migratetype);
2382         }
    		 // 如果是可移动的，那么从CMA区域盗用
2383         if (!page && migratetype == MIGRATE_MOVABLE &&
2384                 gfp_flags & __GFP_CMA)
2385             page = __rmqueue_cma(zone, order);
2386 
    		 // 
2387         if (!page)
2388             page = __rmqueue(zone, order, migratetype, gfp_flags);
2389 
2390         spin_unlock(&zone->lock);
2391         if (!page)
2392             goto failed;
2393         __mod_zone_freepage_state(zone, -(1 << order),
2394                       get_pcppage_migratetype(page));
2395     }
2396 
2397     __mod_zone_page_state(zone, NR_ALLOC_BATCH, -(1 << order));
2398     if (atomic_long_read(&zone->vm_stat[NR_ALLOC_BATCH]) <= 0 &&
2399         !test_bit(ZONE_FAIR_DEPLETED, &zone->flags))
2400         set_bit(ZONE_FAIR_DEPLETED, &zone->flags);
2401 
2402     __count_zone_vm_events(PGALLOC, zone, 1 << order);	//更新本cpu vm event信息
2403     zone_statistics(preferred_zone, zone, gfp_flags);	//更新zone相关信息
2404     local_irq_restore(flags);
2405 
2406     VM_BUG_ON_PAGE(bad_range(zone, page), page);	//如果本页已经本映射，重新分配
2407     return page;
2408 
2409 failed:
2410     local_irq_restore(flags);
2411     return NULL;
2412 }

1901 /*
1902  * Do the hard work of removing an element from the buddy allocator.
1903  * Call me with the zone->lock already held.
1904  */
1905 static struct page *__rmqueue(struct zone *zone, unsigned int order,
1906                 int migratetype, gfp_t gfp_flags)
1907 {
1908     struct page *page;
1909 	
1910     page = __rmqueue_smallest(zone, order, migratetype);
1911     if (unlikely(!page)) {
    		 // 从备用迁移类型盗用页
1912         page = __rmqueue_fallback(zone, order, migratetype);
1913     }
1914 
1915     trace_mm_page_alloc_zone_locked(page, order, migratetype);
1916     return page;
1917 }
```

- 6、慢速分配通道分配内存

```c
3037 static inline struct page *
3038 __alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
3039                         struct alloc_context *ac)
3040 {
3041     bool can_direct_reclaim = gfp_mask & __GFP_DIRECT_RECLAIM;
3042     struct page *page = NULL;
3043     int alloc_flags;
3044     unsigned long pages_reclaimed = 0;
3045     unsigned long did_some_progress;
3046     enum migrate_mode migration_mode = MIGRATE_ASYNC;
3047     bool deferred_compaction = false;
3048     int contended_compaction = COMPACT_CONTENDED_NONE;
3049 
3050     /*
3051      * In the slowpath, we sanity check order to avoid ever trying to
3052      * reclaim >= MAX_ORDER areas which will never succeed. Callers may
3053      * be using allocators in order of preference for an area that is
3054      * too large.
3055      */
3056     if (order >= MAX_ORDER) {
3057         WARN_ON_ONCE(!(gfp_mask & __GFP_NOWARN));
3058         return NULL;
3059     }
3060 
3061     /*
3062      * We also sanity check to catch abuse of atomic reserves being used by
3063      * callers that are not in atomic context.
3064      */
3065     if (WARN_ON_ONCE((gfp_mask & (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)) ==
3066                 (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)))
3067         gfp_mask &= ~__GFP_ATOMIC;
3068 
3069     /*
3070      * If this allocation cannot block and it is for a specific node, then
3071      * fail early.  There's no need to wakeup kswapd or retry for a
3072      * speculative node-specific allocation.
3073      */
3074     if (IS_ENABLED(CONFIG_NUMA) && (gfp_mask & __GFP_THISNODE) && !can_direct_reclaim)
3075         goto nopage;
3076 	 
    	 /*
    	 	异步唤醒kswapd内核线程进行内存回收
    	 */
3077 retry:
3078     if (gfp_mask & __GFP_KSWAPD_RECLAIM)
3079         wake_all_kswapds(order, ac);
3080 
3081     /*
3082      * OK, we're below the kswapd watermark and have kicked background
3083      * reclaim. Now things get more complex, so set up alloc_flags according
3084      * to how we want to proceed.
3085      */
         /*
         	使用low水位获取page失败，异步触发kswapd回收页的同时尝试使用min水位分配页，此处
         	仅仅返回分配标志
         */
3086     alloc_flags = gfp_to_alloc_flags(gfp_mask);
3087 
3088     /*
3089      * Find the true preferred zone if the allocation is unconstrained by
3090      * cpusets.
3091      */
3092     if (!(alloc_flags & ALLOC_CPUSET) && !ac->nodemask) {
3093         struct zoneref *preferred_zoneref;
3094         preferred_zoneref = first_zones_zonelist(ac->zonelist,
3095                 ac->high_zoneidx, NULL, &ac->preferred_zone);
3096         ac->classzone_idx = zonelist_zone_idx(preferred_zoneref);
3097     }
3098 
    	 /*
    	 	使用上面gfp_to_alloc_flags返回的min水位标志接着尝试分配内存
    	 */
3099     /* This is the last chance, in general, before the goto nopage. */
3100     page = get_page_from_freelist(gfp_mask, order,
3101                 alloc_flags & ~ALLOC_NO_WATERMARKS, ac);
3102     if (page) // 分配成功
3103         goto got_pg;
3104 
    	 /*
    	 	上面使用min水位分配失败，接着使用无水位分配
    	 */
3105     /* Allocate without watermarks if the context allows */
3106     if (alloc_flags & ALLOC_NO_WATERMARKS) {
3107         /*
3108          * Ignore mempolicies if ALLOC_NO_WATERMARKS on the grounds
3109          * the allocation is high priority and these type of
3110          * allocations are system rather than user orientated
3111          */
3112         ac->zonelist = node_zonelist(numa_node_id(), gfp_mask);
3113 
3114         page = __alloc_pages_high_priority(gfp_mask, order, ac);
3115 
3116         if (page) { // 分配成功
3117             goto got_pg;
3118         }
3119     }
3120 
         /*
         	上面接着分配失败，判断是否能直接回收内存页
         */
3121     /* Caller is not willing to reclaim, we can't balance anything */
3122     if (!can_direct_reclaim) { 
3123         /*
3124          * All existing users of the deprecated __GFP_NOFAIL are
3125          * blockable, so warn of any new users that actually allow this
3126          * type of allocation to fail.
3127          */
3128         WARN_ON_ONCE(gfp_mask & __GFP_NOFAIL);
3129         goto nopage;
3130     }
3131 
         // 如果进程禁止递归的直接回收页，返回失败
3132     /* Avoid recursion of direct reclaim */
3133     if (current->flags & PF_MEMALLOC)
3134         goto nopage;
3135 
3136     /* Avoid allocations with no watermarks from looping endlessly */
3137     if (test_thread_flag(TIF_MEMDIE) && !(gfp_mask & __GFP_NOFAIL))
3138         goto nopage;
3139 
3140     /*
3141      * Try direct compaction. The first pass is asynchronous. Subsequent
3142      * attempts after direct reclaim are synchronous
3143      */
3144     page = __alloc_pages_direct_compact(gfp_mask, order, alloc_flags, ac,
3145                     migration_mode,
3146                     &contended_compaction,
3147                     &deferred_compaction);
3148     if (page)
3149         goto got_pg;
3150 
3151     /* Checks for THP-specific high-order allocations */
3152     if (is_thp_gfp_mask(gfp_mask)) {
3153         /*
3154          * If compaction is deferred for high-order allocations, it is
3155          * because sync compaction recently failed. If this is the case
3156          * and the caller requested a THP allocation, we do not want
3157          * to heavily disrupt the system, so we fail the allocation
3158          * instead of entering direct reclaim.
3159          */
3160         if (deferred_compaction)
3161             goto nopage;
3162 
3163         /*
3164          * In all zones where compaction was attempted (and not
3165          * deferred or skipped), lock contention has been detected.
3166          * For THP allocation we do not want to disrupt the others
3167          * so we fallback to base pages instead.
3168          */
3169         if (contended_compaction == COMPACT_CONTENDED_LOCK)
3170             goto nopage;
3171 
3172         /*
3173          * If compaction was aborted due to need_resched(), we do not
3174          * want to further increase allocation latency, unless it is
3175          * khugepaged trying to collapse.
3176          */
3177         if (contended_compaction == COMPACT_CONTENDED_SCHED
3178             && !(current->flags & PF_KTHREAD))
3179             goto nopage;
3180     }
3181 
3182     /*
3183      * It can become very expensive to allocate transparent hugepages at
3184      * fault, so use asynchronous memory compaction for THP unless it is
3185      * khugepaged trying to collapse.
3186      */
3187     if (!is_thp_gfp_mask(gfp_mask) || (current->flags & PF_KTHREAD))
3188         migration_mode = MIGRATE_SYNC_LIGHT;
3189 
    	 /*
    	 	直接回收页并分配
    	 */
3190     /* Try direct reclaim and then allocating */
3191     page = __alloc_pages_direct_reclaim(gfp_mask, order, alloc_flags, ac,
3192                             &did_some_progress);
3193     if (page)
3194         goto got_pg;
3195 
3196     /* Do not loop if specifically requested */
3197     if (gfp_mask & __GFP_NORETRY)
3198         goto noretry;
3199 
3200     /* Keep reclaiming pages as long as there is reasonable progress */
3201     pages_reclaimed += did_some_progress;
3202     if ((did_some_progress && order <= PAGE_ALLOC_COSTLY_ORDER) ||
3203         ((gfp_mask & __GFP_REPEAT) && pages_reclaimed < (1 << order))) {
3204         /* Wait for some write requests to complete then retry */
3205         wait_iff_congested(ac->preferred_zone, BLK_RW_ASYNC, HZ/50);
3206         goto retry;
3207     }
3208 
    	 // 触发oom杀进程
3209     /* Reclaim has failed us, start killing things */
3210     page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
3211     if (page)
3212         goto got_pg;
3213 
3214     /* Retry as long as the OOM killer is making progress */
3215     if (did_some_progress) // 判断是否需要重试着唤醒kswapd
3216         goto retry;
3217 
3218 noretry:
3219     /*
3220      * High-order allocations do not necessarily loop after
3221      * direct reclaim and reclaim/compaction depends on compaction
3222      * being called after reclaim so call directly if necessary
3223      */
3224     page = __alloc_pages_direct_compact(gfp_mask, order, alloc_flags,
3225                         ac, migration_mode,
3226                         &contended_compaction,
3227                         &deferred_compaction);
3228     if (page)
3229         goto got_pg;
3230 nopage:
3231     warn_alloc_failed(gfp_mask, order, NULL);
3232 got_pg: // 分配成功，返回page结构
3233     return page;
3234 }
```

6、直接回收页进行分配

```c
2915 /* The really slow allocator path where we enter direct reclaim */
2916 static inline struct page *
2917 __alloc_pages_direct_reclaim(gfp_t gfp_mask, unsigned int order,
2918         int alloc_flags, const struct alloc_context *ac,
2919         unsigned long *did_some_progress)
2920 {   
2921     struct page *page = NULL;
2922     bool drained = false;
2923 	 
     	 // 直接回收
2924     *did_some_progress = __perform_reclaim(gfp_mask, order, ac);
2925     if (unlikely(!(*did_some_progress)))
2926         return NULL;
2927 
    	 // 尝试分配
2928 retry:
2929     page = get_page_from_freelist(gfp_mask, order,
2930                     alloc_flags & ~ALLOC_NO_WATERMARKS, ac);
2931 
2932     /*
2933      * If an allocation failed after direct reclaim, it could be because
2934      * pages are pinned on the per-cpu lists or in high alloc reserves.
2935      * Shrink them them and try again
2936      */
2937     if (!page && !drained) {
2938         unreserve_highatomic_pageblock(ac);
2939         drain_all_pages(NULL);
2940         drained = true;
2941         goto retry;
2942     }
2943 
2944     return page;
2945 }

2886 /* Perform direct synchronous page reclaim */
2887 static int
2888 __perform_reclaim(gfp_t gfp_mask, unsigned int order,
2889                     const struct alloc_context *ac)
2890 {     
2891     struct reclaim_state reclaim_state;
2892     int progress;
2893 
2894     cond_resched();
2895 
2896     /* We now go into synchronous reclaim */
2897     cpuset_memory_pressure_bump();
2898     current->flags |= PF_MEMALLOC;
2899     lockdep_set_current_reclaim_state(gfp_mask);
2900     reclaim_state.reclaimed_slab = 0;
2901     current->reclaim_state = &reclaim_state;
2902     
    	 // 释放page
2903     progress = try_to_free_pages(ac->zonelist, order, gfp_mask,
2904                                 ac->nodemask);
2905 
2906     current->reclaim_state = NULL;
2907     lockdep_clear_current_reclaim_state();
2908     current->flags &= ~PF_MEMALLOC;
2909 
2910     cond_resched();
2911 
2912     return progress;
2913 }

2834 unsigned long try_to_free_pages(struct zonelist *zonelist, int order,
2835                 gfp_t gfp_mask, nodemask_t *nodemask)
2836 {
2837     unsigned long nr_reclaimed;
2838     struct scan_control sc = {
2839         .nr_to_reclaim = SWAP_CLUSTER_MAX,
2840         .gfp_mask = (gfp_mask = memalloc_noio_flags(gfp_mask)),
2841         .order = order,
2842         .nodemask = nodemask,
2843         .priority = DEF_PRIORITY,
2844         .may_writepage = !laptop_mode,
2845         .may_unmap = 1,
2846         .may_swap = 1,
2847     };
2848     
2849     /*
2850      * Do not enter reclaim if fatal signal was delivered while throttled.
2851      * 1 is returned so that the page allocator does not OOM kill at this
2852      * point.
2853      */
2854     if (throttle_direct_reclaim(gfp_mask, zonelist, nodemask))
2855         return 1;
2856 
2857     trace_mm_vmscan_direct_reclaim_begin(order,
2858                 sc.may_writepage,
2859                 gfp_mask);
2860 
2861     nr_reclaimed = do_try_to_free_pages(zonelist, &sc);
2862 
2863     trace_mm_vmscan_direct_reclaim_end(nr_reclaimed);
2864 
2865     return nr_reclaimed;
2866 }

2636 static unsigned long do_try_to_free_pages(struct zonelist *zonelist,
2637                       struct scan_control *sc)
2638 {
2639     int initial_priority = sc->priority;
2640     unsigned long total_scanned = 0;
2641     unsigned long writeback_threshold;
2642     bool zones_reclaimable;
2643 retry:
2644     delayacct_freepages_start();
2645 
2646     if (global_reclaim(sc))
2647         count_vm_event(ALLOCSTALL);
2648 
2649     do {
2650         vmpressure_prio(sc->gfp_mask, sc->target_mem_cgroup,
2651                 sc->priority);
2652         sc->nr_scanned = 0;
    		 /*
    		 	收缩zones，内部调用shrink_zone，这里和kswapd进行内存回收最终调用的函数一样
    		 */
2653         zones_reclaimable = shrink_zones(zonelist, sc);
2654 
2655         total_scanned += sc->nr_scanned;
2656         if (sc->nr_reclaimed >= sc->nr_to_reclaim)
2657             break;
2658 
2659         if (sc->compaction_ready)
2660             break;
2661 
2662         /*
2663          * If we're getting trouble reclaiming, start doing
2664          * writepage even in laptop mode.
2665          */
2666         if (sc->priority < DEF_PRIORITY - 2)
2667             sc->may_writepage = 1;
2668 
2669         /*
2670          * Try to write back as many pages as we just scanned.  This
2671          * tends to cause slow streaming writers to write data to the
2672          * disk smoothly, at the dirtying rate, which is nice.   But
2673          * that's undesirable in laptop mode, where we *want* lumpy
2674          * writeout.  So in laptop mode, write out the whole world.
2675          */
2676         writeback_threshold = sc->nr_to_reclaim + sc->nr_to_reclaim / 2;
2677         if (total_scanned > writeback_threshold) {
2678             wakeup_flusher_threads(laptop_mode ? 0 : total_scanned,
2679                         WB_REASON_TRY_TO_FREE_PAGES);
2680             sc->may_writepage = 1;
2681         }
2682     } while (--sc->priority >= 0);
2683 
2684     delayacct_freepages_end();
2685 
2686     if (sc->nr_reclaimed)
2687         return sc->nr_reclaimed;
2688 
2689     /* Aborted reclaim to try compaction? don't OOM, then */
2690     if (sc->compaction_ready)
2691         return 1;
2692 
2693     /* Untapped cgroup reserves?  Don't OOM, retry. */
2694     if (!sc->may_thrash) {
2695         sc->priority = initial_priority;
2696         sc->may_thrash = 1;
2697         goto retry;
2698     }
2699 
2700     /* Any of the zones still reclaimable?  Don't OOM. */
2701     if (zones_reclaimable)
2702         return 1;
2703 
2704     return 0;
2705 }
```



