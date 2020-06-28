### 1、调用关系

1、alloc_pages(gfp_t gfp_mask, unsigned int order)

2、alloc_pages_current(gfp_mask, order)

3、__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
           				struct zonelist *zonelist, nodemask_t *nodemask)

4、正真分配free  page，buddy的核心

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
3246     int alloc_flags = ALLOC_WMARK_LOW|ALLOC_CPUSET|ALLOC_FAIR;
3247     gfp_t alloc_mask; /* The gfp_t that was actually used for allocation */
3248     struct alloc_context ac = {
3249         .high_zoneidx = gfp_zone(gfp_mask),
3250         .nodemask = nodemask,
3251         .migratetype = gfpflags_to_migratetype(gfp_mask),
3252     };
3253 
3254     gfp_mask &= gfp_allowed_mask;
3255 
3256     lockdep_trace_alloc(gfp_mask);
3257 
3258     might_sleep_if(gfp_mask & __GFP_DIRECT_RECLAIM);
3259 
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
3275     cpuset_mems_cookie = read_mems_allowed_begin();
3276 
3277     /* We set it here, as __alloc_pages_slowpath might have changed it */
3278     ac.zonelist = zonelist;
3279 
3280     /* Dirty zone balancing only done in the fast path */
3281     ac.spread_dirty_pages = (gfp_mask & __GFP_WRITE);
3282 
3283     /* The preferred zone is used for statistics later */
3284     preferred_zoneref = first_zones_zonelist(ac.zonelist, ac.high_zoneidx,
3285                 ac.nodemask ? : &cpuset_current_mems_allowed,
3286                 &ac.preferred_zone);
3287     if (!ac.preferred_zone)
3288         goto out;
3289     ac.classzone_idx = zonelist_zone_idx(preferred_zoneref);
3290 
    		/*
    			首先尝试使用低水位线（zone->water[low]）分配
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

5、慢速分配通道分配内存

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



