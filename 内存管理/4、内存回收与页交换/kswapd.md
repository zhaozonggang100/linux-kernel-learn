reference：

```html
https://blog.csdn.net/weixin_33728268/article/details/86276208?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase

https://blog.csdn.net/weixin_42056635/article/details/87920589
```



源码：

```
mm/vmscan.c
mm/swap.c
```

### 1、kswap源码解析

> 1、kswapd初始化

函数：kswapd_init

文件：vmscan.c

```c
3776 static int __init kswapd_init(void)
3777 { 
3778     int nid;
3779   	 
3780     swap_setup();   
		 /*
		 	每个内存节点（node）创建一个kswapd进程
		 	进程名字：kswapd[node_id]
		 */
3781     for_each_node_state(nid, N_MEMORY)    
3782         kswapd_run(nid);
3783     if (kswapd_cpu_mask == NULL)
3784         hotcpu_notifier(cpu_callback, 0);     
3785     return 0;
3786 }
```

函数：swap_setup

文件：mm/swap.c

```c
1142 /*
1143  * Perform any setup for the swap system
1144  */
/*
   根据物理内存大小设置设定全局量page_cluster,磁盘读道是个费时操作，每次读一个页面过于浪费，
   每次多读几个，这个量就要根据实际物理内存大小来确定
 */
1145 void __init swap_setup(void)
1146 {
1147     unsigned long megs = totalram_pages >> (20 - PAGE_SHIFT);
1148 #ifdef CONFIG_SWAP
1149     int i;
1150 
1151     for (i = 0; i < MAX_SWAPFILES; i++)
1152         spin_lock_init(&swapper_spaces[i].tree_lock);
1153 #endif
1154 
1155     /* Use a smaller cluster for small-memory machines */
1156     if (megs < 16)
1157         page_cluster = 2;
1158     else
1159         page_cluster = 3;
1160     /*
1161      * Right now other parts of the system means that we
1162      * _really_ don't want to cluster much more
1163      */
1164 }
```

函数：kswapd_run

文件：vmscan.c

```c
3736 /*  
3737  * This kswapd start function will be called by init and node-hot-add.
3738  * On node-hot-add, kswapd will moved to proper cpus if cpus are hot-added.
3739  */ 
3740 int kswapd_run(int nid)
3741 {
3742     pg_data_t *pgdat = NODE_DATA(nid);
3743     int ret = 0;
3744     
3745     if (pgdat->kswapd)
3746         return 0;
3747     
		 /*
		 	创建内核线程kswapd
		 		处理函数：kswapd
		 		参数：pg_data_t的内存节点对象（pglist_data）
		 */
3748     pgdat->kswapd = kthread_run(kswapd, pgdat, "kswapd%d", nid);
3749     if (IS_ERR(pgdat->kswapd)) {
3750         /* failure at boot is fatal */
3751         BUG_ON(system_state == SYSTEM_BOOTING);
3752         pr_err("Failed to start kswapd on node %d\n", nid);
3753         ret = PTR_ERR(pgdat->kswapd);
3754         pgdat->kswapd = NULL;
3755     } else if (kswapd_cpu_mask) {
3756         if (set_kswapd_cpu_mask(pgdat))
3757             pr_warn("error setting kswapd cpu affinity mask\n");
3758     }
3759     return ret;
3760 }
```

> 2、核心

kswapd的核心就是kswapd[node_id]内核线程，被周期唤醒检测当前内存水位判断是否进行内存回收

kswpad进程的睡眠条件：看该pgdat下的从0到classzone_idx的zone是否都是balance的

```c
3528 /*
3529  * The background pageout daemon, started as a kernel thread
3530  * from the init process.
3531  *
3532  * This basically trickles out pages so that we have _some_
3533  * free memory available even if there is no other activity
3534  * that frees anything up. This is needed for things like routing
3535  * etc, where we otherwise might have all activity going on in
3536  * asynchronous contexts that cannot page things out.
3537  *
3538  * If there are applications that are active memory-allocators
3539  * (most normal use), this basically shouldn't matter.
3540  */
3541 static int kswapd(void *p)
3542 {
3543     unsigned long order, new_order;
3544     int classzone_idx, new_classzone_idx;
3545     int balanced_classzone_idx;
3546     pg_data_t *pgdat = (pg_data_t*)p;
3547     struct task_struct *tsk = current;
3548 
3549     struct reclaim_state reclaim_state = {
3550         .reclaimed_slab = 0,
3551     };
3552     const struct cpumask *cpumask = cpumask_of_node(pgdat->node_id);
3553 
3554     lockdep_set_current_reclaim_state(GFP_KERNEL);
3555 
3556     if (kswapd_cpu_mask == NULL && !cpumask_empty(cpumask))
3557         set_cpus_allowed_ptr(tsk, cpumask);
3558     current->reclaim_state = &reclaim_state;
3559 
3560     /*
3561      * Tell the memory management that we're a "memory allocator",
3562      * and that if we need more memory we should get access to it
3563      * regardless (see "__alloc_pages()"). "kswapd" should
3564      * never get caught in the normal page freeing logic.
3565      *
3566      * (Kswapd normally doesn't need memory anyway, but sometimes
3567      * you need a small amount of memory in order to be able to
3568      * page out something else, and this flag essentially protects
3569      * us from recursively trying to free more memory as we're
3570      * trying to free the first piece of memory in the first place).
3571      */
3572     tsk->flags |= PF_MEMALLOC | PF_SWAPWRITE | PF_KSWAPD;
3573     set_freezable();
3574 
3575     order = new_order = 0;
3576     classzone_idx = new_classzone_idx = pgdat->nr_zones - 1;
3577     balanced_classzone_idx = classzone_idx;
3578     for ( ; ; ) {
3579         bool ret;
3580 
3581         /*
3582          * While we were reclaiming, there might have been another
3583          * wakeup, so check the values.
3584          */
3585         new_order = pgdat->kswapd_max_order;
3586         new_classzone_idx = pgdat->classzone_idx;
3587         pgdat->kswapd_max_order =  0;
3588         pgdat->classzone_idx = pgdat->nr_zones - 1;
3589 
3590         if (order < new_order || classzone_idx > new_classzone_idx) {
3591             /*
3592              * Don't sleep if someone wants a larger 'order'
3593              * allocation or has tigher zone constraints
3594              */
3595             order = new_order;
3596             classzone_idx = new_classzone_idx;
3597         } else {
    			 /*
    			 	尝试休眠kswapd进程
    			 */
3598             kswapd_try_to_sleep(pgdat, order, classzone_idx,
3599                         balanced_classzone_idx);
3600             order = pgdat->kswapd_max_order;
3601             classzone_idx = pgdat->classzone_idx;
3602             new_order = order;
3603             new_classzone_idx = classzone_idx;
3604             pgdat->kswapd_max_order = 0;
3605             pgdat->classzone_idx = pgdat->nr_zones - 1;
3606         }
3607 
    		 /*
    		 	尝试休眠kswapd[node_id]进程
    		 */
3608         ret = try_to_freeze();
3609         if (kthread_should_stop())
3610             break;
3611 
3612         /*
3613          * We can speed up thawing tasks if we don't call balance_pgdat
3614          * after returning from the refrigerator
3615          */
3616         if (!ret) { // 上面freeze  kswapd进程失败，需要进行页面回收
3617             trace_mm_vmscan_kswapd_wake(pgdat->node_id, order);
    			 /*
    			 	下面函数会进行页面的释放，保证被选中的node的所有zone的水位都达到high
    			 	zone的high水位：high_wmark_pages(zone)，是回收页面的核心函数
    			 */
3618             balanced_classzone_idx = balance_pgdat(pgdat, order,
3619                                 classzone_idx);
3620         }
3621     }
3622 
3623     tsk->flags &= ~(PF_MEMALLOC | PF_SWAPWRITE | PF_KSWAPD);
3624     current->reclaim_state = NULL;
3625     lockdep_clear_current_reclaim_state();
3627     return 0;
3628 }
```

> 3、回收页面过程

vmscan.c

```c
https://blog.csdn.net/dog250/article/details/5303562
3288 /*
3289  * For kswapd, balance_pgdat() will work across all this node's zones until
3290  * they are all at high_wmark_pages(zone).
3291  *
3292  * Returns the highest zone idx kswapd was reclaiming at
3293  *
3294  * There is special handling here for zones which are full of pinned pages.
3295  * This can happen if the pages are all mlocked, or if they are all used by
3296  * device drivers (say, ZONE_DMA).  Or if they are all in use by hugetlb.
3297  * What we do is to detect the case where all pages in the zone have been
3298  * scanned twice and there has been zero successful reclaim.  Mark the zone as
3299  * dead and from now on, only perform a short scan.  Basically we're polling
3300  * the zone for when the problem goes away.
3301  *
3302  * kswapd scans the zones in the highmem->normal->dma direction.  It skips
3303  * zones which have free_pages > high_wmark_pages(zone), but once a zone is
3304  * found to have free_pages <= high_wmark_pages(zone), we scan that zone and the
3305  * lower zones regardless of the number of free pages in the lower zones. This
3306  * interoperates with the page allocator fallback scheme to ensure that aging
3307  * of pages is balanced across the zones.
3308  */
3309 static int balance_pgdat(pg_data_t *pgdat, int order, int classzone_idx)
3310 {
3311     int i;
3312     int end_zone = 0;   /* Inclusive.  0 = ZONE_DMA */
3313     unsigned long nr_soft_reclaimed;
3314     unsigned long nr_soft_scanned;
3315     struct scan_control sc = {
3316         .gfp_mask = GFP_KERNEL,
3317         .order = order,
3318         .priority = DEF_PRIORITY,
3319         .may_writepage = !laptop_mode,
3320         .may_unmap = 1,
3321         .may_swap = 1,
3322     };
3323     count_vm_event(PAGEOUTRUN);
3324 
    	 /*
    	 	循环遍历node中的zone直到有一个zone是blance的
    	 */
3325     do {
3326         bool raise_priority = true;
3327         unsigned long lru_pages = 0;
3328 
3329         sc.nr_reclaimed = 0;
3330 
3331         /*
3332          * Scan in the highmem->dma direction for the highest
3333          * zone which needs scanning
3334          */
3335         for (i = pgdat->nr_zones - 1; i >= 0; i--) {
3336             struct zone *zone = pgdat->node_zones + i;
3337 
3338             if (!populated_zone(zone))
3339                 continue;
3340 
3341             if (sc.priority != DEF_PRIORITY &&
3342                 !zone_reclaimable(zone))
3343                 continue;
3344 
3345             /*
3346              * Do some background aging of the anon list, to give
3347              * pages a chance to be referenced before reclaiming.
3348              */
3349             age_active_anon(zone, &sc);
3350 
3351             /*
3352              * If the number of buffer_heads in the machine
3353              * exceeds the maximum allowed level and this node
3354              * has a highmem zone, force kswapd to reclaim from
3355              * it to relieve lowmem pressure.
3356              */
3357             if (buffer_heads_over_limit && is_highmem_idx(i)) {
3358                 end_zone = i;
3359                 break;
3360             }
3361 
3362             if (!zone_balanced(zone, order, false, 0, 0)) {
    				 // 找到blanced的zone，break
3363                 end_zone = i;
3364                 break;
3365             } else {
3366                 /*
3367                  * If balanced, clear the dirty and congested
3368                  * flags
3369                  */
3370                 clear_bit(ZONE_CONGESTED, &zone->flags);
3371                 clear_bit(ZONE_DIRTY, &zone->flags);
3372             }
3373         }
3374 
3375         if (i < 0)
3376             goto out;
3377 
3378         /*
3379          * If we're getting trouble reclaiming, start doing writepage
3380          * even in laptop mode.
3381          */
3382         if (sc.priority < DEF_PRIORITY - 2)
3383             sc.may_writepage = 1;
3384 
3385         for (i = 0; i <= end_zone; i++) {
3386             struct zone *zone = pgdat->node_zones + i;
3387 
3388             if (!populated_zone(zone))
3389                 continue;
3390 
3391             lru_pages += zone_reclaimable_pages(zone);
3392         }
3393 
3394         /*
3395          * Now scan the zone in the dma->highmem direction, stopping
3396          * at the last zone which needs scanning.
3397          *
3398          * We do this because the page allocator works in the opposite
3399          * direction.  This prevents the page allocator from allocating
3400          * pages behind kswapd's direction of progress, which would
3401          * cause too much scanning of the lower zones.
3402          */
3403         for (i = 0; i <= end_zone; i++) {
3404             struct zone *zone = pgdat->node_zones + i;
3405 
3406             if (!populated_zone(zone))
3407                 continue;
3408 
3409             if (sc.priority != DEF_PRIORITY &&
3410                 !zone_reclaimable(zone))
3411                 continue;
3412 
3413             sc.nr_scanned = 0;
3414 
3415             nr_soft_scanned = 0;
3416             /*
3417              * Call soft limit reclaim before calling shrink_zone.
3418              */
3419             nr_soft_reclaimed = mem_cgroup_soft_limit_reclaim(zone,
3420                             order, sc.gfp_mask,
3421                             &nr_soft_scanned);
3422             sc.nr_reclaimed += nr_soft_reclaimed;
3423 
3424             /*
3425              * There should be no need to raise the scanning
3426              * priority if enough pages are already being scanned
3427              * that that high watermark would be met at 100%
3428              * efficiency.
3429              */
    			 /*
    			 	这里开始收缩zone
    			 */
3430             if (kswapd_shrink_zone(zone, end_zone, &sc, lru_pages))
3431                 raise_priority = false;
3432         }
3433 
3434         /*
3435          * If the low watermark is met there is no need for processes
3436          * to be throttled on pfmemalloc_wait as they should not be
3437          * able to safely make forward progress. Wake them
3438          */
3439         if (waitqueue_active(&pgdat->pfmemalloc_wait) &&
3440                 pfmemalloc_watermark_ok(pgdat))
3441             wake_up_all(&pgdat->pfmemalloc_wait);
3442 
3443         /* Check if kswapd should be suspending */
3444         if (try_to_freeze() || kthread_should_stop())
3445             break;
3446 
3447         /*
3448          * Raise priority if scanning rate is too low or there was no
3449          * progress in reclaiming pages
3450          */
3451         if (raise_priority || !sc.nr_reclaimed)
3452             sc.priority--;
3453     } while (sc.priority >= 1 &&
3454             !pgdat_balanced(pgdat, order, classzone_idx));
3455 
3456 out:
3457     /*
3458      * Return the highest zone idx we were reclaiming at so
3459      * prepare_kswapd_sleep() makes the same decisions as here.
3460      */
3461     return end_zone;
3462 }
```

> 4、调用函数kswapd_shrink_zone收缩zone

```c
3230 /*
3231  * kswapd shrinks the zone by the number of pages required to reach
3232  * the high watermark.
3233  *
3234  * Returns true if kswapd scanned at least the requested number of pages to
3235  * reclaim or if the lack of progress was due to pages under writeback.
3236  * This is used to determine if the scanning priority needs to be raised.
3237  */
3238 static bool kswapd_shrink_zone(struct zone *zone,
3239                    int classzone_idx,
3240                    struct scan_control *sc,
3241                 unsigned long lru_pages)
3242 {
3243     unsigned long balance_gap;
3244     bool lowmem_pressure;
3245 
3246     /* Reclaim above the high watermark. */
3247     sc->nr_to_reclaim = max(SWAP_CLUSTER_MAX, high_wmark_pages(zone));
3248 
3249     /*
3250      * We put equal pressure on every zone, unless one zone has way too
3251      * many pages free already. The "too many pages" is defined as the
3252      * high wmark plus a "gap" where the gap is either the low
3253      * watermark or 1% of the zone, whichever is smaller.
3254      */
3255     balance_gap = min(low_wmark_pages(zone), DIV_ROUND_UP(
3256             zone->managed_pages, KSWAPD_ZONE_BALANCE_GAP_RATIO));
3257 
3258     /*
3259      * If there is no low memory pressure or the zone is balanced then no
3260      * reclaim is necessary
3261      */
3262     lowmem_pressure = (buffer_heads_over_limit && is_highmem(zone));
3263     if (!lowmem_pressure && zone_balanced(zone, sc->order, false,
3264                         balance_gap, classzone_idx))
3265         return true;
3266 
3267     shrink_zone(zone, sc, zone_idx(zone) == classzone_idx);
    	 /*
    	 	遍历lmk（lowmemorykiller）通过register_shrink注册的杀进程函数
    	 */
3268     shrink_slab_lmk(sc->gfp_mask, zone_to_nid(zone), NULL,
3269             sc->nr_scanned, lru_pages);
3270 
3271     clear_bit(ZONE_WRITEBACK, &zone->flags);
3272 
3273     /*
3274      * If a zone reaches its high watermark, consider it to be no longer
3275      * congested. It's possible there are dirty pages backed by congested
3276      * BDIs but as pressure is relieved, speculatively avoid congestion
3277      * waits.
3278      */
3279     if (zone_reclaimable(zone) &&
3280         zone_balanced(zone, sc->order, false, 0, classzone_idx)) {
3281         clear_bit(ZONE_CONGESTED, &zone->flags);
3282         clear_bit(ZONE_DIRTY, &zone->flags);
3283     }
3284 
3285     return sc->nr_scanned >= sc->nr_to_reclaim;
3286 }
```

