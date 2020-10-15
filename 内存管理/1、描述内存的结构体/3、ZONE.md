### 1、内核数据结构  
include/linux/mmzone.h（kernel-4.4.138）

```c
 332 struct zone {
 333     /* Read-mostly fields */
 334 
 335     /* zone watermarks, access with *_wmark_pages(zone) macros */
 336     unsigned long watermark[NR_WMARK];
 337 
 338     unsigned long nr_reserved_highatomic;
 339 
 340     /*
 341      * We don't know if the memory that we're going to allocate will be
 342      * freeable or/and it will be released eventually, so to avoid totally
 343      * wasting several GB of ram we must reserve some of the lower zone
 344      * memory (otherwise we risk to run OOM on the lower zones despite
 345      * there being tons of freeable ram on the higher zones).  This array is
 346      * recalculated at runtime if the sysctl_lowmem_reserve_ratio sysctl
 347      * changes.
 348      */
     	 /*
     	 	参考：
				https://blog.csdn.net/rikeyone/article/details/85037297   
				https://zhuanlan.zhihu.com/p/81961211  

				在每个zone中会保留一部分内存，作为关键分配需求，为了避免在内存不足时发生OOM，
				这个保留的内存是runtime修改的，而且与该zone中ram总大小有关系  

				当用户态修改/proc/sys/vm/lowmem_reserve_ratio中各个zone的值时内核会调用		
				setup_per_zone_lowmem_reserve用来设置zone的lowmem_reserve值  

				处理过程：lowmem_reserve_ratio_sysctl_handler--
				>setup_per_zone_lowmem_reserve  
				kernel/sysctl.c中指定了proc目录项对应的修改操作函数  
				init_per_zone_wmark_min也会调用setup_per_zone_lowmem_reserve设置
				lowmem_reserve数组中每个zone的lowmem_reserve大小  
     	 */
 349     long lowmem_reserve[MAX_NR_ZONES];
 350 
 351 #ifdef CONFIG_NUMA
 352     int node;  
 353 #endif
 354 
 355     /*
 356      * The target ratio of ACTIVE_ANON to INACTIVE_ANON pages on
 357      * this zone's LRU.  Maintained by the pageout code.
 358      */
 359     unsigned int inactive_ratio;
 360 
 361     struct pglist_data  *zone_pgdat;
     	 /*
     	 	per-cpu高速缓存，减少自旋锁的竞争
     	 */
 362     struct per_cpu_pageset __percpu *pageset;
 363
 364     /*
 365      * This is a per-zone reserve of pages that should not be
 366      * considered dirtyable memory.
 367      */
 368     unsigned long       dirty_balance_reserve;
 369 
 370 #ifndef CONFIG_SPARSEMEM
 371     /*
 372      * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
 373      * In SPARSEMEM, this map is stored in struct mem_section
 374      */
 375     unsigned long       *pageblock_flags;
 376 #endif /* CONFIG_SPARSEMEM */
 377 
 378 #ifdef CONFIG_NUMA
 379     /*
 380      * zone reclaim becomes active if more unmapped pages exist.
 381      */
 382     unsigned long       min_unmapped_pages;
 383     unsigned long       min_slab_pages;
 384 #endif /* CONFIG_NUMA */
 385 
 386     /* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
         // zone的第一个页框号-+
 387     unsigned long       zone_start_pfn;
 388 
 389     /*
 390      * spanned_pages is the total pages spanned by the zone, including
 391      * holes, which is calculated as:
 392      *  spanned_pages = zone_end_pfn - zone_start_pfn;
 393      *
 394      * present_pages is physical pages existing within the zone, which
 395      * is calculated as:
 396      *  present_pages = spanned_pages - absent_pages(pages in holes);
 397      *
 398      * managed_pages is present pages managed by the buddy system, which
 399      * is calculated as (reserved_pages includes pages allocated by the
 400      * bootmem allocator):
 401      *  managed_pages = present_pages - reserved_pages;
 402      *
 403      * So present_pages may be used by memory hotplug or memory power
 404      * management logic to figure out unmanaged pages by checking
 405      * (present_pages - managed_pages). And managed_pages should be used
 406      * by page allocator and vm scanner to calculate all kinds of watermarks
 407      * and thresholds.
 408      *
 409      * Locking rules:
 410      *
 411      * zone_start_pfn and spanned_pages are protected by span_seqlock.
 412      * It is a seqlock because it has to be read outside of zone->lock,
 413      * and it is done in the main allocator path.  But, it is written
 414      * quite infrequently.
 415      *
 416      * The span_seq lock is declared along with zone->lock because it is
 417      * frequently read in proximity to zone->lock.  It's good to
 418      * give them a chance of being in the same cacheline.
 419      *
 420      * Write access to present_pages at runtime should be protected by
 421      * mem_hotplug_begin/end(). Any reader who can't tolerant drift of
 422      * present_pages should get_online_mems() to get a stable value.
 423      *
 424      * Read access to managed_pages should be safe because it's unsigned
  425      * long. Write access to zone->managed_pages and totalram_pages are
 426      * protected by managed_page_count_lock at runtime. Idealy only
 427      * adjust_managed_page_count() should be used instead of directly
 428      * touching zone->managed_pages and totalram_pages.
 429      */
 430     unsigned long       managed_pages;	// 该zone中被buddy管理的页不包括bootmem allocator分配器分配的页
 431     unsigned long       spanned_pages;	// 该zone管理的所有页包括空洞
 432     unsigned long       present_pages;	// 该zone包括的所有页不包含空洞
 433 
 434     const char      *name;
 435 
 436 #ifdef CONFIG_MEMORY_ISOLATION
 437     /*
 438      * Number of isolated pageblock. It is used to solve incorrect
 439      * freepage counting problem due to racy retrieving migratetype
 440      * of pageblock. Protected by zone->lock.
 441      */
 442     unsigned long       nr_isolate_pageblock;
 443 #endif
 444 
 445 #ifdef CONFIG_MEMORY_HOTPLUG
 446     /* see spanned/present_pages for more description */
 447     seqlock_t       span_seqlock;
 448 #endif
 449 
 450     /*
 451      * wait_table       -- the array holding the hash table
 452      * wait_table_hash_nr_entries   -- the size of the hash table array
 453      * wait_table_bits  -- wait_table_size == (1 << wait_table_bits)
 454      *
  455      * The purpose of all these is to keep track of the people
 456      * waiting for a page to become available and make them
 457      * runnable again when possible. The trouble is that this
 458      * consumes a lot of space, especially when so few things
 459      * wait on pages at a given time. So instead of using
 460      * per-page waitqueues, we use a waitqueue hash table.
 461      *
 462      * The bucket discipline is to sleep on the same queue when
 463      * colliding and wake all in that wait queue when removing.
 464      * When something wakes, it must check to be sure its page is
 465      * truly available, a la thundering herd. The cost of a
 466      * collision is great, but given the expected load of the
 467      * table, they should be so rare as to be outweighed by the
 468      * benefits from the saved space.
 469      *
 470      * __wait_on_page_locked() and unlock_page() in mm/filemap.c, are the
 471      * primary users of these fields, and in mm/page_alloc.c
 472      * free_area_init_core() performs the initialization of them.
 473      */
 474     wait_queue_head_t   *wait_table;	// 避免在zone所在节点上内存不足时进程分配内存阻塞在等待队列上被唤醒后出现惊群现象，该域是node中zone分配页面等待队列的哈希表
 475     unsigned long       wait_table_hash_nr_entries;
 476     unsigned long       wait_table_bits;
 477 
     	 /*
     	 	在zone结构中包含了spinlock锁，为了保证多核竞争锁，使用cacheline对齐，
     	 	保证锁占用一个独立的cacheline，这样的实现代价是以空间换时间  
     	 */
 478     ZONE_PADDING(_pad1_)
 479     /* free areas of different sizes */
         //11,空闲页框的组织结构
 480     struct free_area    free_area[MAX_ORDER];
 481 
 482     /* zone flags, see below */
 483     unsigned long       flags;
 484 
 485     /* Write-intensive fields used from the page allocator */
 486     spinlock_t      lock;
 487 
 488     ZONE_PADDING(_pad2_)
 489 
 490     /* Write-intensive fields used by page reclaim */
 491 
 492     /* Fields commonly accessed by the page reclaim scanner */
 493     spinlock_t      lru_lock;
     	 /*
     	 	一个node中的每个zone都有一个lruvec的lru向量用来管理最近不常使用的物理页
     	 	配合kswap进程回收页
     	 */
 494     struct lruvec       lruvec;
 495 
 496     /* Evictions & activations on the inactive file list */
 497     atomic_long_t       inactive_age;
 498 
 499     /*
 500      * When free pages are below this point, additional steps are taken
 501      * when reading the number of free pages to avoid per-cpu counter
 502      * drift allowing watermarks to be breached
 503      */
 504     unsigned long percpu_drift_mark;
 505 
     /*
     	CONFIG_COMPACTION：内存压缩  
		CONFIG_CMA：连续物理内存管理，比如专门用来DMA  
     */
 506 #if defined CONFIG_COMPACTION || defined CONFIG_CMA
 507     /* pfn where compaction free scanner should start */
 508     unsigned long       compact_cached_free_pfn;
 509     /* pfn where async and sync compaction migration scanner should start */
 510     unsigned long       compact_cached_migrate_pfn[2];
 511 #endif
 512 
 513 #ifdef CONFIG_COMPACTION
 514     /*
 515      * On compaction failure, 1<<compact_defer_shift compactions
 516      * are skipped before trying again. The number attempted since
 517      * last failure is tracked with compact_considered.
 518      */
 519     unsigned int        compact_considered;
 520     unsigned int        compact_defer_shift;
 521     int         compact_order_failed;
 522 #endif
 523 
 524 #if defined CONFIG_COMPACTION || defined CONFIG_CMA
 525     /* Set to true when the PG_migrate_skip bits should be cleared */
 526     bool            compact_blockskip_flush;
 527 #endif
 528 
 529     ZONE_PADDING(_pad3_)
 530     /* Zone statistics */
 531     atomic_long_t       vm_stat[NR_VM_ZONE_STAT_ITEMS];
 532 } ____cacheline_internodealigned_in_smp;

 /*
 	kswapd进程回收内存时扫描的page双向链表
 */
 217 struct lruvec {
 218     struct list_head lists[NR_LRU_LISTS];	// 五个链表
 219     struct zone_reclaim_stat reclaim_stat;
 220 #ifdef CONFIG_MEMCG
 221     struct zone *zone;
 222 #endif
 223 };

 176 enum lru_list {
 177     LRU_INACTIVE_ANON = LRU_BASE, 							// 0
 178     LRU_ACTIVE_ANON = LRU_BASE + LRU_ACTIVE,				// 1
 179     LRU_INACTIVE_FILE = LRU_BASE + LRU_FILE,				// 2
 180     LRU_ACTIVE_FILE = LRU_BASE + LRU_FILE + LRU_ACTIVE,    // 3
 181     LRU_UNEVICTABLE,										// 4
 182     NR_LRU_LISTS											// 5
 183 };
```

### 2、ZONE的种类  

- ZONE_DMA  

	ISA机器上受限于DMA外设为16MB，因系统不同区域大小不同，DMA外设不受限时为0    

- ZONE_DMA32   

	64位系统上为了兼容32位DMA外设，32位系统上该区域为0        

- ZONE_NORMAL（必须包含）  

	直接映射区  

- ZONE_HIGHMEM   

	32位系统为了访问大于1GB的物理内存，64位系统上不存在  

- ZONE_MOVABLE   

	为了支持内存热插拔（memory hotplug），可迁移的内存不一定在该区域，但该区域中的内存一定可迁移  
	逻辑热插拔：对于虚拟机内存按需分配  
	物理热插拔：对于NUMA服务器的支持，不需要就设置位offline，降低功耗可以解决内存碎片化   

- ZONE_DEVICE    

	支持热插拔的设备  


查看系统NODE包含的zone的类型：cat /proc/pagetypeinfo    

### 3、ZONE中包含的物理页  

- 1、每CPU高速缓存  

	struct per_cpu_pageset __percpu * pageset；

- 2、buddy系统管理的内存  

	struct free_area free_area[MAX_ORDER]；