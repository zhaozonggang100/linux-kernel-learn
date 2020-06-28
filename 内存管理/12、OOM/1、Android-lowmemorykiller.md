

reference：

http://gityuan.com/2016/09/17/android-lowmemorykiller/

https://www.jianshu.com/p/4dbe9bbe0449

https://blog.csdn.net/u014175785/article/details/85050046

https://www.jianshu.com/p/519544843d8c

### 1、概述

Android通过内核提供的drivers/staging/android/lowmemorykiller.c驱动接口来实现内存不足时kill Android创建的进程。

Android中zygote孵化出来的进程都由ActivityManagerService管理，ActivityManagerService会调用lmkd将进程的oom_score_adj更新到proc中，接着进入内核执行进程扫描杀进程。

### 2、lowmemorykiller框架

lowmemorykiller主要由三部分组成

1、AMS部分的ProcessList（framework）

2、Native进程lmkd（native）

3、内核中的LowMemoryKiller部分（kernel）



framework层的ProcessList和native层的lmkd通过socket（/dev/socket/lmkd）通信

lmkd和内核中的LowMemoryKiller通过writeFileString向文件节点写内容方法进行通信



lmkd杀进程方式：

1、将阀值写入文件节点发送给内核的LowMemoryKiller，由内核进行杀进程处理（4.12中已经移除）

2、通过cgroup监控内存使用情况，自行计算杀掉进程。



> 为了防止剩余内存过低，Android在内核空间有lowmemorykiller(简称LMK)，LMK是通过注册shrinker来触发低内存回收的，这个机制并不太优雅，可能会拖慢Shrinkers内存扫描速度，已从内核4.12中移除，后续会采用用户空间的LMKD + memory cgroups机制。



### 3、LMKD如何处理来自framework层的连接以及cmd

1、创建socket

2、epoll监听socket，排除中断和信号的干扰

3、调用ctrl_data_handler



> framework发送给lmkd的cmd：

1、cmd_target 调整最小内存阀值和adj值

会将两个值写入到文件系统节点：

​	/sys/module/lowmemorykiller/parameters/minfree		18432, 23040, 27648, 32256, 55296, 80640

​	/sys/module/lowmemorykiller/parameters/adj				 0,		  100,	  200,	  300,	 900,     906

2、cmd_procprio 调整进程的adj值

调整进程的被杀优先级（/proc/pid/oom_score_adj）

3、cmd_procremove 移除对应的进程

由内核处理



```
lmdk源码：system/core/lmkd/lmkd.c
```



### 4、kernel  lowmemorykiller处理

```
源码：drivers/staging/android/lowmemorykiller.c
历史：Google也在2018年4月在system/core/lmkd仓库中添加README.md，并宣判了lowmemorykiller的死期——kernel-4.12

kernel-4.4.138：
131 enum zone_stat_item {
 132     /* First 128 byte cacheline (assuming 64 bit words) */
 133     NR_FREE_PAGES,
 134     NR_ALLOC_BATCH,
 135     NR_LRU_BASE,
 136     NR_INACTIVE_ANON = NR_LRU_BASE, /* must match order of LRU_[IN]ACTIVE */
 137     NR_ACTIVE_ANON,     /*  "     "     "   "       "         */
 138     NR_INACTIVE_FILE,   /*  "     "     "   "       "         */
 139     NR_ACTIVE_FILE,     /*  "     "     "   "       "         */
 140     NR_UNEVICTABLE,     /*  "     "     "   "       "         */
 141     NR_MLOCK,       /* mlock()ed pages found and moved off LRU */
 142     NR_ANON_PAGES,  /* Mapped anonymous pages */
 143     NR_FILE_MAPPED, /* pagecache pages mapped into pagetables.
 144                only modified from process context */
 145     NR_FILE_PAGES,
 146     NR_FILE_DIRTY,
 147     NR_WRITEBACK,
 148     NR_SLAB_RECLAIMABLE,
 149     NR_SLAB_UNRECLAIMABLE,
 150     NR_PAGETABLE,       /* used for pagetables */
 151     /* Second 128 byte cacheline */
 152     NR_KERNEL_STACK,
 153     NR_KAISERTABLE,
 154     NR_UNSTABLE_NFS,    /* NFS unstable pages */
 155     NR_BOUNCE,
 156     NR_VMSCAN_WRITE,
 157     NR_VMSCAN_IMMEDIATE,    /* Prioritise for reclaim when writeback ends */
 158     NR_WRITEBACK_TEMP,  /* Writeback using temporary buffers */
 159     NR_ISOLATED_ANON,   /* Temporary isolated pages from anon lru */
 160     NR_ISOLATED_FILE,   /* Temporary isolated pages from file lru */
 161     NR_SHMEM,       /* shmem pages (included tmpfs/GEM pages) */
 162     NR_DIRTIED,     /* page dirtyings since bootup */
 163     NR_WRITTEN,     /* page writings since bootup */
 164     NR_PAGES_SCANNED,   /* pages scanned since last reclaim */
 165 #ifdef CONFIG_NUMA
 166     NUMA_HIT,       /* allocated in intended node */
 167     NUMA_MISS,      /* allocated in non intended node */
 168     NUMA_FOREIGN,       /* was intended here, hit elsewhere */
 169     NUMA_INTERLEAVE_HIT,    /* interleaver preferred this zone */
 170     NUMA_LOCAL,     /* allocation from local node */
 171     NUMA_OTHER,     /* allocation from other node */
 172 #endif
 173     WORKINGSET_REFAULT,
 174     WORKINGSET_ACTIVATE,
 175     WORKINGSET_NODERECLAIM,
 176     NR_ANON_TRANSPARENT_HUGEPAGES,
 177     NR_FREE_CMA_PAGES,
 178     NR_SWAPCACHE,
 179     NR_INDIRECTLY_RECLAIMABLE_BYTES, /* measured in bytes */
 180     NR_VM_ZONE_STAT_ITEMS 
 181};


 66 static short lowmem_adj[6] = {
 67     0,
 68     1,
 69     6,
 70     12,
 71 };  
 72 static int lowmem_adj_size = 4;
 73 static int lowmem_minfree[6] = {
 74     3 * 512,    /* 6MB */
 75     2 * 1024,   /* 8MB */
 76     4 * 1024,   /* 16MB */
 77     16 * 1024,  /* 64MB */
 78 };
 79 static int lowmem_minfree_size = 4;
 80 static int lmk_fast_run = 1;

407 static unsigned long lowmem_scan(struct shrinker *s, struct shrink_control *sc)
408 {
409     struct task_struct *tsk;                        
410     struct task_struct *selected = NULL;        // 最终被选中要kill的进程
411     unsigned long rem = 0;                      // 归还给系统的内存
412     int tasksize;
413     int i;
414     int ret = 0;
415     short min_score_adj = OOM_SCORE_ADJ_MAX + 1;  // 1000 + 1 
416     int minfree = 0;
417     int selected_tasksize = 0;                  // 被选中进程占用内存大小
418     short selected_oom_score_adj;               // 被选中进程的oom_score_adj值
419     int array_size = ARRAY_SIZE(lowmem_adj);
420     int other_free;
421     int other_file;
422 
423     if (!mutex_trylock(&scan_mutex))
424         return 0;
425 
        /*
            global_page_state可以获取当前系统可用的（剩余）内存大小
            NR_FREE_PAGES是zone_stat_item的枚举类型，node的每个zone都定义了的atomic_long_t       vm_stat[NR_VM_ZONE_STAT_ITEMS];
            代表当前zone包含的所有page的不同属性页的数量
            同时kernel还定义了全局的atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS] __cacheline_aligned_in_smp;
        */
426     other_free = global_page_state(NR_FREE_PAGES);
427 
        /*
            下面是计算可以回收的page（有后背file的page，tmpfs的NR_SHEMEM和zcache的NR_UNEVICTABLE：代表不可回收）
        */
428     if (global_page_state(NR_SHMEM) + total_swapcache_pages() <
429         global_page_state(NR_FILE_PAGES) + zcache_pages())      // tmpfs占用的page cache小于整个page cache
430         other_file = global_page_state(NR_FILE_PAGES) + zcache_pages() -
431                         global_page_state(NR_SHMEM) -
432                         global_page_state(NR_UNEVICTABLE) -
433                         total_swapcache_pages();
434     else
435         other_file = 0;
436 
        /*
            和native的lmkd有关系，可能是memory cgroup
        */
437     tune_lmk_param(&other_free, &other_file, sc);
438 
        /*
            让lowmem_minfree和lowmem_adj对应，array_size一样长
            for循环开始找到other_free（剩余的空闲内存页）大小在lowmem_minfree中的哪个区间，从而找到lowmem_adj数组对应的项
            min_score_adj = lowmem_adj[i]，作为被select进程的oom_score_adj参考
        */
439     if (lowmem_adj_size < array_size)
440         array_size = lowmem_adj_size;
441     if (lowmem_minfree_size < array_size)
442         array_size = lowmem_minfree_size;
443     for (i = 0; i < array_size; i++) {
444         minfree = lowmem_minfree[i];    // 通过lowmem_minfree数组项的值和当前剩余内存的值做对比
445         if (other_free < minfree && other_file < minfree) {
446             min_score_adj = lowmem_adj[i];
447             break;
448         }
449     }
450 
451     ret = adjust_minadj(&min_score_adj);  // 
452 
453     lowmem_print(3, "lowmem_scan %lu, %x, ofree %d %d, ma %hd\n",
454             sc->nr_to_scan, sc->gfp_mask, other_free,
455             other_file, min_score_adj);
456 
457     if (min_score_adj == OOM_SCORE_ADJ_MAX + 1) {   // 满足条件说明不需要进行内存回收
458         trace_almk_shrink(0, ret, other_free, other_file, 0);
459         lowmem_print(5, "lowmem_scan %lu, %x, return 0\n",
460                  sc->nr_to_scan, sc->gfp_mask);
461         mutex_unlock(&scan_mutex);
462         return 0;
463     }
464 
465     selected_oom_score_adj = min_score_adj;     // 被选中的进程的oom值
467     rcu_read_lock();

        /*
            根据最小adj遍历所有进程，找到oom_score_adj大于adj的进程，如果有多个进程的adj相等，选择进程rss（独占内存+共享库大小）最大的
        */
468     for_each_process(tsk) {
469         struct task_struct *p;
470         short oom_score_adj;
471 
472         if (tsk->flags & PF_KTHREAD)    // 过滤内核线程
473             continue;
474 
475         /* if task no longer has any memory ignore it */
476         if (test_task_flag(tsk, TIF_MM_RELEASED))   // 进程的内存已经释放
477             continue;
478 
479         if (time_before_eq(jiffies, lowmem_deathpending_timeout)) {
480             if (test_task_flag(tsk, TIF_MEMDIE)) {
481                 rcu_read_unlock();
482                 mutex_unlock(&scan_mutex);
483                 return 0;
484             }
485         }
486 
487         p = find_lock_task_mm(tsk);
488         if (!p)
489             continue;
490 
491         oom_score_adj = p->signal->oom_score_adj;
492         if (oom_score_adj < min_score_adj) {  // 如果当前进程的oom_score_adj比最小的adj还小不杀，adj越高代表越优先被杀
493             task_unlock(p);
494             continue;
495         }
            // 获取进程的占用内存大小(rss值)，也就是进程独占内存 + 共享库大小
496         tasksize = get_mm_rss(p->mm);
497         task_unlock(p);
498         if (tasksize <= 0)
499             continue;
500         if (selected) { // 只有找到一个小于最小adj的进程selected才会被设置，比较当前选中的进程adj和之前选中进程的adj和rss
501             if (oom_score_adj < selected_oom_score_adj)
502                 continue;
503             if (oom_score_adj == selected_oom_score_adj &&
504                 tasksize <= selected_tasksize)
505                 continue;
506         }
507         selected = p;
508         selected_tasksize = tasksize;
509         selected_oom_score_adj = oom_score_adj;
510         lowmem_print(3, "select '%s' (%d), adj %hd, size %d, to kill\n",
511                  p->comm, p->pid, oom_score_adj, tasksize);
512     }
513     if (selected) {   // 找到要kill的进程
514         long cache_size, cache_limit, free;
515 
516         if (test_task_flag(selected, TIF_MEMDIE) &&
517             (test_task_state(selected, TASK_UNINTERRUPTIBLE))) {
518             lowmem_print(2, "'%s' (%d) is already killed\n",
519                      selected->comm,
520                      selected->pid);
521             rcu_read_unlock();
522             mutex_unlock(&scan_mutex);
523             return 0;
524         }
525 
526         task_lock(selected);
527         send_sig(SIGKILL, selected, 0);   // 发送信号杀进程
/*
529          * FIXME: lowmemorykiller shouldn't abuse global OOM killer
530          * infrastructure. There is no real reason why the selected
531          * task should have access to the memory reserves.
532          */
533         if (selected->mm)
534             mark_oom_victim(selected);
535         task_unlock(selected);
536         cache_size = other_file * (long)(PAGE_SIZE / 1024);     // 剩余内存的页转换为kb
537         cache_limit = minfree * (long)(PAGE_SIZE / 1024);       // 上面找到的lowmem_minfree项
538         free = other_free * (long)(PAGE_SIZE / 1024);
539         trace_lowmemory_kill(selected, cache_size, cache_limit, free);
540         lowmem_print(1, "Killing '%s' (%d) (tgid %d), adj %hd,\n" \
541                     "   to free %ldkB on behalf of '%s' (%d) because\n" \
542                     "   cache %ldkB is below limit %ldkB for oom_score_adj %hd\n" \
543                 "   Free memory is %ldkB above reserved.\n" \
544                 "   Free CMA is %ldkB\n" \
545                 "   Total reserve is %ldkB\n" \
546                 "   Total free pages is %ldkB\n" \
547                 "   Total file cache is %ldkB\n" \
548                 "   Total zcache is %ldkB\n" \
549                 "   GFP mask is 0x%x\n",
550                  selected->comm, selected->pid, selected->tgid,
551                  selected_oom_score_adj,
552                  selected_tasksize * (long)(PAGE_SIZE / 1024),
553                  current->comm, current->pid,
554                  cache_size, cache_limit,
555                  min_score_adj,
556                  free,
557                  global_page_state(NR_FREE_CMA_PAGES) *
558                 (long)(PAGE_SIZE / 1024),
559                  totalreserve_pages * (long)(PAGE_SIZE / 1024),
560                  global_page_state(NR_FREE_PAGES) *
561                 (long)(PAGE_SIZE / 1024),
562                  global_page_state(NR_FILE_PAGES) *
563                 (long)(PAGE_SIZE / 1024),
564                  (long)zcache_pages() * (long)(PAGE_SIZE / 1024),
565                  sc->gfp_mask);
566 
567         if (lowmem_debug_level >= 2 && selected_oom_score_adj == 0) {
568             show_mem(SHOW_MEM_FILTER_NODES);
569             dump_tasks(NULL, NULL);
570         }
571 
572         lowmem_deathpending_timeout = jiffies + HZ;
573         rem += selected_tasksize;
574         rcu_read_unlock();
575         /* give the system time to free up the memory */
576         msleep_interruptible(20);
577         trace_almk_shrink(selected_tasksize, ret,
578                   other_free, other_file,
579                   selected_oom_score_adj);
580     } else {
581         trace_almk_shrink(1, ret, other_free, other_file, 0);
582         rcu_read_unlock();
583     }
584 
585     lowmem_print(4, "lowmem_scan %lu, %x, return %lu\n",
586              sc->nr_to_scan, sc->gfp_mask, rem);
587     mutex_unlock(&scan_mutex);
588     return rem;
589}
```

### 5、从kswapd到lowmem_scan

1、kswapd_init

2、kswapd_run

3、kswapd：每个内存节点的kswpad线程，周期性唤醒检测是否需要回收内存

4、balance_pgdat

5、kswapd_shrink_zone

6、shrink_slab_lmk

7、do_shrink_slab

8、shrinker->scan_objects(shrinker, shrinkctl)