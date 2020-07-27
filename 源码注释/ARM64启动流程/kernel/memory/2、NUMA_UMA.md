### 1、概述

概念性的东西大家可以CSDN上找找，此处针对源码讲解讲解代码的实现

1、NUMA提出了内存簇的概念，cpu访问不同的的内存簇的代价是不同的，离cpu越近，效率越高，Linux内核使用`struct pg_data_t`来表示一块内存簇，也叫内存节点。

```c
// 1、如果内核config配置了多节点，系统就会有一个全局的pgdat_list的数组（基于2.6内核的深入理解Linux内核虚拟内存写的是链表，但是我在源码的pg_data_t结构中没有发现有next相关的成员）
/*
	file：arch/ia64/mm/discontig.c
	整个系统只有ia64架构中定义了多node的结构
	可能ARM64上要实现多node需要有一些自己的工作量
	这里我们暂时先讨论只有一个NODE的NUMA架构，其实只是数据结构表示的不同，其余和UMA差不多把？？欢迎一起讨论
*/
pg_data_t *pgdat_list[MAX_NUMNODES];

// 2、如果配置为只有一个节点，NUMA即使只有一个节点也需要用pg_data_t来表示整个Linux管理的虚拟内存
/*
	file：mm/bootmem.c
*/
24 #ifndef CONFIG_NEED_MULTIPLE_NODES	// 查看自己系统是否是单节点，看config中该选项是否打开
25 struct pglist_data __refdata contig_page_data = {
26     .bdata = &bootmem_node_data[0]
27 };
28 EXPORT_SYMBOL(contig_page_data);
29 #endif

/*
	file：mm/nobootmem.c
	我的config里是没有定义CONFIG_NEED_MULTIPLE_NODES，说明只有一个NODE，那么下面的静态定义变量就代表当前系统的内存节点了
	在memory_init.md中我们会介绍Linux如何初始化该节点的
*/
30 #ifndef CONFIG_NEED_MULTIPLE_NODES
31 struct pglist_data __refdata contig_page_data;
32 EXPORT_SYMBOL(contig_page_data);
33 #endif
```



2、每个内存节点内又根据虚拟地址的不同，划分了不同的ZONE，每一个zone用`struct zone`表示

### 2、数据结构

1、node节点

```c
/*
	file：include/linux/mmzone.h
*/ 
623 /*
 624  * On NUMA machines, each NUMA node would have a pg_data_t to describe
 625  * it's memory layout. On UMA machines there is a single pglist_data which
 626  * describes the whole memory.
 627  *
 628  * Memory statistics and page replacement data structures are maintained on a
 629  * per-zone basis.
 630  */
 631 struct bootmem_data;
 632 typedef struct pglist_data {
 633     struct zone node_zones[MAX_NR_ZONES];
 634     struct zonelist node_zonelists[MAX_ZONELISTS];
 635     int nr_zones;
 636 #ifdef CONFIG_FLAT_NODE_MEM_MAP /* means !SPARSEMEM */
 637     struct page *node_mem_map;
 638 #ifdef CONFIG_PAGE_EXTENSION
 639     struct page_ext *node_page_ext;
 640 #endif
 641 #endif
 642 #ifndef CONFIG_NO_BOOTMEM			// 我所用的源码打开该配置选项，所以这部分不会编译
 643     struct bootmem_data *bdata;
 644 #endif
 645 #ifdef CONFIG_MEMORY_HOTPLUG
 646     /*
 647      * Must be held any time you expect node_start_pfn, node_present_pages
 648      * or node_spanned_pages stay constant.  Holding this will also
 649      * guarantee that any pfn_valid() stays that way.
 650      *
 651      * pgdat_resize_lock() and pgdat_resize_unlock() are provided to
 652      * manipulate node_size_lock without checking for CONFIG_MEMORY_HOTPLUG.
 653      *
 654      * Nests above zone->lock and zone->span_seqlock
 655      */
 656     spinlock_t node_size_lock;
 657 #endif
 658     unsigned long node_start_pfn;
 659     unsigned long node_present_pages; /* total number of physical pages */
 660     unsigned long node_spanned_pages; /* total size of physical page
 661                          range, including holes */
 662     int node_id;
 663     wait_queue_head_t kswapd_wait;
 664     wait_queue_head_t pfmemalloc_wait;
 665     struct task_struct *kswapd; /* Protected by
 666                        mem_hotplug_begin/end() */
 667     int kswapd_order;
 668     enum zone_type kswapd_classzone_idx;
 669 
 670     int kswapd_failures;        /* Number of 'reclaimed == 0' runs */
 671 
 672 #ifdef CONFIG_COMPACTION
 673     int kcompactd_max_order;
 674     enum zone_type kcompactd_classzone_idx;
 675     wait_queue_head_t kcompactd_wait;
 676     struct task_struct *kcompactd;
 677 #endif
 678 #ifdef CONFIG_NUMA_BALANCING
 679     /* Lock serializing the migrate rate limiting window */
 680     spinlock_t numabalancing_migrate_lock;
 681 
 682     /* Rate limiting time interval */
 683     unsigned long numabalancing_migrate_next_window;
 684 
 685     /* Number of pages migrated during the rate limiting time interval */
 686     unsigned long numabalancing_migrate_nr_pages;
 687 #endif
 688     /*
 689      * This is a per-node reserve of pages that are not available
 690      * to userspace allocations.
 691      */
 692     unsigned long       totalreserve_pages;
 693 
 694 #ifdef CONFIG_NUMA
 695     /*
 696      * zone reclaim becomes active if more unmapped pages exist.
 697      */
 698     unsigned long       min_unmapped_pages;
 699     unsigned long       min_slab_pages;
 700 #endif /* CONFIG_NUMA */
 701 
 702     /* Write-intensive fields used by page reclaim */
 703     ZONE_PADDING(_pad1_)
 704     spinlock_t      lru_lock;
 705 
 706 #ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
 707     /*
 708      * If memory initialisation on large machines is deferred then this
 709      * is the first PFN that needs to be initialised.
 710      */
 711     unsigned long first_deferred_pfn;
 712     /* Number of non-deferred pages */
 713     unsigned long static_init_pgcnt;
 714 #endif /* CONFIG_DEFERRED_STRUCT_PAGE_INIT */
 715 
 716 #ifdef CONFIG_TRANSPARENT_HUGEPAGE
 717     spinlock_t split_queue_lock;
 718     struct list_head split_queue;
 719     unsigned long split_queue_len;
 720 #endif
 721 
 722     /* Fields commonly accessed by the page reclaim scanner */
 723     struct lruvec       lruvec;
 724 
 725     /*
 726      * The target ratio of ACTIVE_ANON to INACTIVE_ANON pages on
 727      * this node's LRU.  Maintained by the pageout code.
 728      */
 729     unsigned int inactive_ratio;
 730 
 731     unsigned long       flags;
 732 
 733     ZONE_PADDING(_pad2_)
 734 
 735     /* Per-node vmstats */
 736     struct per_cpu_nodestat __percpu *per_cpu_nodestats;
 737     atomic_long_t       vm_stat[NR_VM_NODE_STAT_ITEMS];
 738 } pg_data_t;
```





