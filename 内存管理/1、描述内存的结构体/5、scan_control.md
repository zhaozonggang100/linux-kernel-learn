### 1、概述

当分配内存不足时调用kswapd内核线程或者直接内存回收最终调用shrink_zone函数进行内存回收需要向低层传递scan_control结构体，告诉低层应该如何回收内存

### 2、数据结构

vmscan.c

```c
61 struct scan_control {
  62     /* How many pages shrink_list() should reclaim */
         // 应该回收多少页，执行直接回收时应该是32
  63     unsigned long nr_to_reclaim;
  64 
  65     /* This context's GFP mask */
  66     gfp_t gfp_mask;
  67 
  68     /* Allocation order */
  69     int order;
  70 
  71     /*
  72      * Nodemask of nodes allowed by the caller. If NULL, all nodes
  73      * are scanned.
  74      */
         // 调用者分配内存允许使用的内存节点，如果null，可以使用所有节点
  75     nodemask_t  *nodemask;
  76 
  77     /*
  78      * The memory cgroup that hit its limit and as a result is the
  79      * primary target of this reclaim invocation.
  80      */
  81     struct mem_cgroup *target_mem_cgroup;
  82 
      	 /*
      	 	扫描优先级，一次扫描的页数是（LRU链表的页的总和 >> 扫描优先级），默认初始值为12
      	 */
  83     /* Scan (total_size >> priority) pages at once */
  84     int priority;
  85 
         // 是否允许把修改过的页写回到存储设备
  86     unsigned int may_writepage:1; 
  87 
         // 是否允许回收页表映射了的物理页
  88     /* Can mapped pages be reclaimed? */
  89     unsigned int may_unmap:1;
  90 
      	 // 是否允许把匿名页换出到交换分区
  91     /* Can pages be swapped as part of reclaim? */
  92     unsigned int may_swap:1;
  93 
  94     /* Can cgroups be reclaimed below their normal consumption range? */
  95     unsigned int may_thrash:1;
  96 
  97     unsigned int hibernation_mode:1;
  98 
  99     /* One of the zones is ready for compaction */
 100     unsigned int compaction_ready:1;
 101 
         // 报告扫描过的不活动页面的数量
 102     /* Incremented by the number of inactive pages that were scanned */
 103     unsigned long nr_scanned;
 104 
     	 // 报告回收了多少页
 105     /* Number of pages freed so far during a call to shrink_zones() */
 106     unsigned long nr_reclaimed;
 107 };
```

