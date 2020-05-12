### 1、内核数据结构  
`struct zone（include/linux/mmzone.h）`

### 2、成员  
- 1、ZONE_PADDING  

	在zone结构中包含了spinlock锁，为了保证多核竞争锁，使用cacheline对齐，保证锁占用一个独立的cacheline，这样的实现代价是以空间换时间  

- 2、defined CONFIG_COMPACTION || defined CONFIG_CMA  

	CONFIG_COMPACTION：内存压缩  
	CONFIG_CMA：连续物理内存管理，比如专门用来DMA  

- 3、long lowmem_reserve[MAX_NR_ZONES]  

	参考：
	https://blog.csdn.net/rikeyone/article/details/85037297   
	https://zhuanlan.zhihu.com/p/81961211  
	
	在每个zone中会保留一部分内存，作为关键分配需求，为了避免在内存不足时发生OOM，这个保留的内存是runtime修改的，而且与该zone中ram总大小有关系  
	
	当用户态修改/proc/sys/vm/lowmem_reserve_ratio中各个zone的值时内核会调用setup_per_zone_lowmem_reserve用来设置zone的lowmem_reserve值  
	处理过程：lowmem_reserve_ratio_sysctl_handler-->setup_per_zone_lowmem_reserve  
	kernel/sysctl.c中指定了proc目录项对应的修改操作函数  
	init_per_zone_wmark_min也会调用setup_per_zone_lowmem_reserve设置lowmem_reserve数组中每个zone的lowmem_reserve大小  

- 4、struct per_cpu_pageset __percpu *pageset  

```
per-cpu高速缓存
```

- 5、三种页  
```
unsigned long       managed_pages;			// 该zone管理的页，present_pages - reserved_pages
unsigned long       spanned_pages;			//  该zone占据的所有页，包含空洞，spanned_pages = zone_end_pfn - zone_start_pfn
unsigned long       present_pages;			// 该zone包含的页，不包含空洞，present_pages = spanned_pages - absent_pages(pages in holes)
```
### 3、ZONE的种类  

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

### 4、ZONE中包含的物理页  

- 1、每CPU高速缓存  

	struct per_cpu_pageset __percpu * pageset；

- 2、buddy系统管理的内存  

	struct free_area free_area[MAX_ORDER]；