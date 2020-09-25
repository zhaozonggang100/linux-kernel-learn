参考：

```html
https://www.cnblogs.com/linhaostudy/p/10058492.html
https://zhuanlan.zhihu.com/p/207001939?utm_source=wechat_timeline
https://blog.csdn.net/rikeyone/article/details/86496229
https://blog.csdn.net/juS3Ve/article/details/80656701（内存模型）
http://www.voidcn.com/article/p-qtnltdel-up.html（CONFIG_SPARSEMEM_VMEMMAP）
https://blog.csdn.net/ooonebook/article/details/52690132（vmlinux.ld.S）
https://www.cnblogs.com/arnoldlu/p/10526814.html(memblock)
https://blog.csdn.net/snail_coder/article/details/94383185(memblock1)
```



### 1、kernel如何知道内存的大小

一般的内存的地址空间会在DTS中定义，而内核初始化时会去解析设备树获取内存的地址空间

初始化函数调用流程

```c
/* start_kernel
-->setup_arch(&command_line)	// arch/arm64/kernel/setup.c
-->setup_machine_fdt(__fdt_pointer) // arch/arm64/kernel/setup.c
-->early_init_dt_scan(dt_virt) // arch/arm64/kernel/setup.c
-->early_init_dt_scan_nodes() // drivers/of/fdt.c
-->early_init_dt_scan_memory // drivers/of/fdt.c
*/

 992 /**
 993  * early_init_dt_scan_memory - Look for and parse memory nodes
 994  */
 995 int __init early_init_dt_scan_memory(unsigned long node, const char *uname,
 996                      int depth, void *data)
 997 {
 998     const char *type = of_get_flat_dt_prop(node, "device_type", NULL);
 999     const __be32 *reg, *endp;
1000     int l;
1001     bool hotpluggable;
1002 
1003     /* We are scanning "memory" nodes only */
1004     if (type == NULL || strcmp(type, "memory") != 0)
1005         return 0;
1006    
1007     reg = of_get_flat_dt_prop(node, "linux,usable-memory", &l);
1008     if (reg == NULL)
1009         reg = of_get_flat_dt_prop(node, "reg", &l);
1010     if (reg == NULL)
1011         return 0;
1012 
1013     endp = reg + (l / sizeof(__be32));
1014     hotpluggable = of_get_flat_dt_prop(node, "hotpluggable", NULL);
1015 
1016     pr_debug("memory scan node %s, reg size %d,\n", uname, l);
1017 
1018     while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {
1019         u64 base, size;
1020 
1021         base = dt_mem_next_cell(dt_root_addr_cells, &reg);
1022         size = dt_mem_next_cell(dt_root_size_cells, &reg);
1023 
1024         if (size == 0)
1025             continue;
1026         pr_debug(" - %llx ,  %llx\n", (unsigned long long)base,
1027             (unsigned long long)size);
1028 
    		 // 将从设备树获取的内存的起始地址和大小添加到memblock子系统中
1029         early_init_dt_add_memory_arch(base, size);
1030 
1031         if (!hotpluggable)
1032             continue;
1033 
1034         if (early_init_dt_mark_hotplug_memory_arch(base, size))
1035             pr_warn("failed to mark hotplug range 0x%llx - 0x%llx\n",
1036                 base, base + size);
1037     }
1038 
1039     return 0;
1040 }
```



###  2、在kernel的内存管理初始化之前要分配内存怎么办

```c
// 为了在kernel初始化内存管理之前使用内存，kernel实现了memblock机制来使用物理内存，当buddy初始化完成之后所有内存交由buddy管理

/*
	在初始化过程中, 还必须建立内存管理的数据结构, 以及很多事务. 因为内核在内存管理完全初始化之前就需要使用内存. 在系统启动过程期间, 使用了额外的简化悉尼股市的内存管理模块, 然后在初始完成后将旧的模块丢掉
	这个阶段的内存分配其实很简单, 因此我们往往称之为内存分配器(而不是内存管理器), 早期的内核中内存分配器使用的bootmem引导分配器, 它基于一个内存位图bitmap, 使用最优适配算法来查找内存, 但是这个分配器有很大的缺陷, 最严重的就是内存碎片的问题, 因此在后来的内核中将其舍弃《而使用了新的memblock机制. memblock机制的初始化在arm64上是通过arm64_memblock_init函数来实现的
*/

// 调用：start_kernel->setup_arch->arm64_memblock_init
302 void __init arm64_memblock_init(void)
303 {
304     const s64 linear_region_size = -(s64)PAGE_OFFSET;
305 
306     /* Handle linux,usable-memory-range property */
307     fdt_enforce_memory_region();
308 
309     /* Remove memory above our supported physical address size */
310     memblock_remove(1ULL << PHYS_MASK_SHIFT, ULLONG_MAX);
311 
312     /*
313      * Ensure that the linear region takes up exactly half of the kernel
314      * virtual address space. This way, we can distinguish a linear address
315      * from a kernel/module/vmalloc address by testing a single bit.
316      */
317     BUILD_BUG_ON(linear_region_size != BIT(VA_BITS - 1));
318 
319     /*
320      * Select a suitable value for the base of physical memory.
321      */
322     memstart_addr = round_down(memblock_start_of_DRAM(),
323                    ARM64_MEMSTART_ALIGN);
324 
325     /*
326      * Remove the memory that we will not be able to cover with the
327      * linear mapping. Take care not to clip the kernel which may be
328      * high in memory.
329      */
330     memblock_remove(max_t(u64, memstart_addr + linear_region_size,
331             __pa_symbol(_end)), ULLONG_MAX);
332     if (memstart_addr + linear_region_size < memblock_end_of_DRAM()) {
333         /* ensure that memstart_addr remains sufficiently aligned */
334         memstart_addr = round_up(memblock_end_of_DRAM() - linear_region_size,
335                      ARM64_MEMSTART_ALIGN);
336         memblock_remove(0, memstart_addr);
337     }
338 
339     /*
340      * Apply the memory limit if it was set. Since the kernel may be loaded
341      * high up in memory, add back the kernel region that must be accessible
342      * via the linear mapping.
343      */
344     if (memory_limit != PHYS_ADDR_MAX) {
345         memblock_mem_limit_remove_map(memory_limit);
346         memblock_add(__pa_symbol(_text), (u64)(_end - _text));
347     }
348 
349     if (IS_ENABLED(CONFIG_BLK_DEV_INITRD) && phys_initrd_size) {
350         /*
351          * Add back the memory we just removed if it results in the
352          * initrd to become inaccessible via the linear mapping.
353          * Otherwise, this is a no-op
354          */
355         u64 base = phys_initrd_start & PAGE_MASK;
356         u64 size = PAGE_ALIGN(phys_initrd_start + phys_initrd_size) - base;
357 
358         /*
359          * We can only add back the initrd memory if we don't end up
360          * with more memory than we can address via the linear mapping.
361          * It is up to the bootloader to position the kernel and the
362          * initrd reasonably close to each other (i.e., within 32 GB of
363          * each other) so that all granule/#levels combinations can
364          * always access both.
365          */
366         if (WARN(base < memblock_start_of_DRAM() ||
367              base + size > memblock_start_of_DRAM() +
368                        linear_region_size,
369             "initrd not fully accessible via the linear mapping -- please check your bootloader ...\n")) {
370             phys_initrd_size = 0;
371         } else {
372             memblock_remove(base, size); /* clear MEMBLOCK_ flags */
373             memblock_add(base, size);
374             memblock_reserve(base, size);
375         }
376     }
377 
378     if (IS_ENABLED(CONFIG_RANDOMIZE_BASE)) {
379         extern u16 memstart_offset_seed;
380         u64 range = linear_region_size -
381                 (memblock_end_of_DRAM() - memblock_start_of_DRAM());
382 
383         /*
384          * If the size of the linear region exceeds, by a sufficient
385          * margin, the size of the region that the available physical
386          * memory spans, randomize the linear region as well.
387          */
388         if (memstart_offset_seed > 0 && range >= ARM64_MEMSTART_ALIGN) {
389             range /= ARM64_MEMSTART_ALIGN;
390             memstart_addr -= ARM64_MEMSTART_ALIGN *
391                      ((range * memstart_offset_seed) >> 16);
392         }
393     }
394 
395     /*
396      * Register the kernel text, kernel data, initrd, and initial
397      * pagetables with memblock.
398      */
399     memblock_reserve(__pa_symbol(_text), _end - _text);
400     if (IS_ENABLED(CONFIG_BLK_DEV_INITRD) && phys_initrd_size) {
401         /* the generic initrd code expects virtual addresses */
402         initrd_start = __phys_to_virt(phys_initrd_start);
403         initrd_end = initrd_start + phys_initrd_size;
404     }
405 
406     early_init_fdt_scan_reserved_mem();
407 
408     /* 4GB maximum for 32-bit only capable devices */
409     if (IS_ENABLED(CONFIG_ZONE_DMA32))
410         arm64_dma_phys_limit = max_zone_dma_phys();
411     else
412         arm64_dma_phys_limit = PHYS_MASK + 1;
413 
414     reserve_crashkernel();
415 
416     reserve_elfcorehdr();
417 
418     high_memory = __va(memblock_end_of_DRAM() - 1) + 1;
419 
420     dma_contiguous_reserve(arm64_dma_phys_limit);
421 }
```



### 3、kernel如何使用物理内存，默认使能MMU，cpu访问的都是虚拟地址，虚拟地址需要经过MMU查找页表映射到对应的物理地址，所以需要初始化完memblock之后接着初始化页表项

- 1、首先内核需要为物理内存建立页表项

```c
// arm64去除了高端内存，创建页表的过程
// start_kernel-->setup_arch(&command_line)-->paging_init()-->map_mem(pgdp)

 // file:arch/arm64/mm/mmu.c
 /*
 	paging_init建立的页表只能内核空间使用，用户空间创建进程时创建自己的页表
 */
 667 void __init paging_init(void)
 668 {
 669     pgd_t *pgdp = pgd_set_fixmap(__pa_symbol(swapper_pg_dir));
 670 
 671     map_kernel(pgdp);
 672     map_mem(pgdp);
 673 
 674     pgd_clear_fixmap();
 675 
 676     cpu_replace_ttbr1(lm_alias(swapper_pg_dir));
 677     init_mm.pgd = swapper_pg_dir;
 678 
 679     memblock_free(__pa_symbol(init_pg_dir),
 680               __pa_symbol(init_pg_end) - __pa_symbol(init_pg_dir));
 681 
 682     memblock_allow_resize();
 683 }

 458 static void __init map_mem(pgd_t *pgdp)
 459 {
 460     phys_addr_t kernel_start = __pa_symbol(_text);
 461     phys_addr_t kernel_end = __pa_symbol(__init_begin);
 462     struct memblock_region *reg;
 463     int flags = 0;
 464 
 465     if (rodata_full || debug_pagealloc_enabled())
 466         flags = NO_BLOCK_MAPPINGS | NO_CONT_MAPPINGS;
 467     
 468     /*
 469      * Take care not to create a writable alias for the
 470      * read-only text and rodata sections of the kernel image.
 471      * So temporarily mark them as NOMAP to skip mappings in
 472      * the following for-loop
 473      */
 474     memblock_mark_nomap(kernel_start, kernel_end - kernel_start);
 475 #ifdef CONFIG_KEXEC_CORE
 476     if (crashk_res.end)
 477         memblock_mark_nomap(crashk_res.start,
 478                     resource_size(&crashk_res));
 479 #endif
 480 
 481     /* map all the memory banks */
 482     for_each_memblock(memory, reg) {
 483         phys_addr_t start = reg->base;
 484         phys_addr_t end = start + reg->size;
 485 
 486         if (start >= end)
 487             break;
 488         if (memblock_is_nomap(reg))
 489             continue;
 490 
 491         __map_memblock(pgdp, start, end, PAGE_KERNEL, flags);
 492     }
 493 
  494     /*
 495      * Map the linear alias of the [_text, __init_begin) interval
 496      * as non-executable now, and remove the write permission in
 497      * mark_linear_text_alias_ro() below (which will be called after
 498      * alternative patching has completed). This makes the contents
 499      * of the region accessible to subsystems such as hibernate,
 500      * but protects it from inadvertent modification or execution.
 501      * Note that contiguous mappings cannot be remapped in this way,
 502      * so we should avoid them here.
 503      */
 504     __map_memblock(pgdp, kernel_start, kernel_end,
 505                PAGE_KERNEL, NO_CONT_MAPPINGS);
 506     memblock_clear_nomap(kernel_start, kernel_end - kernel_start);
 507 
 508 #ifdef CONFIG_KEXEC_CORE
 509     /*
 510      * Use page-level mappings here so that we can shrink the region
 511      * in page granularity and put back unused memory to buddy system
 512      * through /sys/kernel/kexec_crash_size interface.
 513      */
 514     if (crashk_res.end) {
 515         __map_memblock(pgdp, crashk_res.start, crashk_res.end + 1,
 516                    PAGE_KERNEL,
 517                    NO_BLOCK_MAPPINGS | NO_CONT_MAPPINGS);
 518         memblock_clear_nomap(crashk_res.start,
 519                      resource_size(&crashk_res));
 520     }
 521 #endif
 522 }
```



- 2、在使用内存之前需要先清空页表一级映射，ARM64为PGD



### 4、页表初始化完成了，但是如何知道哪些虚拟页被映射了以及映射到的物理页地址，内核提供了buddy子系统用来管理所有的page，所以接下来就是内存管理buddy相关数据结构的初始化

```c
// 一般的嵌入式arm64平台只有一个node，有多个zone
// node-zone的初始化在bootmem_init中进行
// start_kernel-->setup_arch-->bootmem_init

/*
	file:arch/arm64/mm/init.c
	功能：
		1、始化内存数据结构包括内存节点, 内存域和页帧page
		2、从这里开始了与体系无关的数据结构
*/
423 void __init bootmem_init(void)
424 {
425     unsigned long min, max;
426 
    	/*
    		下面从memblock中获取最大和最小页帧号，为后面buddy中zone管理的页帧做准备
    	*/
427     min = PFN_UP(memblock_start_of_DRAM());
428     max = PFN_DOWN(memblock_end_of_DRAM());
429 
430     early_memtest(min << PAGE_SHIFT, max << PAGE_SHIFT);
431         
432     max_pfn = max_low_pfn = max;
433     min_low_pfn = min;
434 
    	/*
    		支持numa架构
    	*/
435     arm64_numa_init();
436     /*
437      * Sparsemem tries to allocate bootmem in memory_present(), so must be
438      * done after the fixed reservations.
439      */
440     memblocks_present();
441     
442     sparse_init();
    	// 来初始化节点和管理区的一些数据项，从这个函数开始内存的初始化与体系无关
443     zone_sizes_init(min, max);
444 
        // 初始化完成, 显示memblock的保留的所有内存信息
445     memblock_dump_all();
446 }


/*
	zone和node相关结构的初始化
	函数：zone_sizes_init
	详解：
		1、由于arm64支持numa和uma两张架构，根据config中的CONFIG_NUMA来执行不同的过程
		2、ARM64中没有高端内存，所以zone到NORMAL（MOVABLE和DEVICE暂部考虑）
*/
177 #ifdef CONFIG_NUMA	// 定义了NUMA
178 
179 static void __init zone_sizes_init(unsigned long min, unsigned long max)
180 {
    	// 系统中各个内存域的页帧边界，保存在max_zone_pfn数组
    	// 早期的内核还需记录各结点页帧的分配情况，保存在全局变量early_node_map中
    	// 设置各内存zone的最大页帧号
181     unsigned long max_zone_pfns[MAX_NR_ZONES]  = {0};
182 
183 #ifdef CONFIG_ZONE_DMA32
    	// 如果定义了CONFIG_ZONE_DMA32，所有node的ZONE_DMA32的最大页帧号是PFN_DOWN(max_zone_dma_phys()
184     max_zone_pfns[ZONE_DMA32] = PFN_DOWN(max_zone_dma_phys());
185 #endif
    	// 所有node的NORMAL区域的最大页帧号是max，也就是从memblock分配器获取到的最大页帧号
186     max_zone_pfns[ZONE_NORMAL] = max;
187 
    	/*
    		有多个node，则使用下面函数初始化所有node的管理数据结构
    		max_zone_pfns数组在arm64中不存在高端内存
    	*/
188     free_area_init_nodes(max_zone_pfns);
189 }
190 
191 #else	// 没有定义NUMA
192 
193 static void __init zone_sizes_init(unsigned long min, unsigned long max)
194 {
195     struct memblock_region *reg;
196     unsigned long zone_size[MAX_NR_ZONES], zhole_size[MAX_NR_ZONES];
197     unsigned long max_dma = min;
198 
199     memset(zone_size, 0, sizeof(zone_size));
200 
201     /* 4GB maximum for 32-bit only capable devices */
202 #ifdef CONFIG_ZONE_DMA32
203     max_dma = PFN_DOWN(arm64_dma_phys_limit);
204     zone_size[ZONE_DMA32] = max_dma - min;
205 #endif
206     zone_size[ZONE_NORMAL] = max - max_dma;
207 
208     memcpy(zhole_size, zone_size, sizeof(zhole_size));
209 
210     for_each_memblock(memory, reg) {
211         unsigned long start = memblock_region_memory_base_pfn(reg);
212         unsigned long end = memblock_region_memory_end_pfn(reg);
213 
214         if (start >= max)
215             continue;
216 
217 #ifdef CONFIG_ZONE_DMA32
218         if (start < max_dma) {
219             unsigned long dma_end = min(end, max_dma);
220             zhole_size[ZONE_DMA32] -= dma_end - start;
221         }
222 #endif
223         if (end > max_dma) {
224             unsigned long normal_end = min(end, max);
225             unsigned long normal_start = max(start, max_dma);
226             zhole_size[ZONE_NORMAL] -= normal_end - normal_start;
227         }
228     }
229 
        // 初始化内存节点，UMA就一个节点
230     free_area_init_node(0, zone_size, min, zhole_size);
231 }
232 
233 #endif /* CONFIG_NUMA */
    
    
7288 /**
7289  * free_area_init_nodes - Initialise all pg_data_t and zone data
7290  * @max_zone_pfn: an array of max PFNs for each zone
7291  *
7292  * This will call free_area_init_node() for each active node in the system.
7293  * Using the page ranges provided by memblock_set_node(), the size of each
7294  * zone in each node and their holes is calculated. If the maximum PFN
7295  * between two adjacent zones match, it is assumed that the zone is empty.
7296  * For example, if arch_max_dma_pfn == arch_max_dma32_pfn, it is assumed
7297  * that arch_max_dma32_pfn has no pages. It is also assumed that a zone
7298  * starts where the previous one ended. For example, ZONE_DMA32 starts
7299  * at arch_max_dma_pfn.
7300  */
/*
	file：mm/page_alloc.c
	功能：free_area_init_nodes初始化了NUMA系统中所有结点的pg_data_t和zone、page的数据, 并打印了管理区信息
	CONFIG_NUMA、CONFIG_NODES_SHIFT定义了是否开启NUMA和node个数
*/
7301 void __init free_area_init_nodes(unsigned long *max_zone_pfn)
7302 {
7303     unsigned long start_pfn, end_pfn;
7304     int i, nid;
7305 
7306     /* Record where the zone boundaries are */
    	 /* 
    	 	全局数组arch_zone_lowest_possible_pfn用来存储各个内存域可使用的最低内存
    	 	页帧编号   	
    	 */
7307     memset(arch_zone_lowest_possible_pfn, 0,
7308                 sizeof(arch_zone_lowest_possible_pfn));
    	 /*
    	 	全局数组arch_zone_highest_possible_pfn用来存储各个内存域可使用的最高内存页
    	 	帧编号
    	 */
7309     memset(arch_zone_highest_possible_pfn, 0,
7310                 sizeof(arch_zone_highest_possible_pfn));
7311 
    	 /*
    	 	 辅助函数find_min_pfn_with_active_regions用于找到注册的最低内存域中可用的
    	 	 编号最小的页帧，最小的页帧是memblock子系统设置的
    	 */
7312     start_pfn = find_min_pfn_with_active_regions();
7313 
    	 /*依次遍历，确定各个内存域的边界*/
7314     for (i = 0; i < MAX_NR_ZONES; i++) {
    		 /*
    		 	由于ZONE_MOVABLE是一个虚拟内存域不与真正的硬件内存域关联该内存域的边界总是
    		 	设置为0
    		 */
7315         if (i == ZONE_MOVABLE)
7316             continue;
7317 		
    		 // max_zone_pfn记录了各个内存域包含的最大页帧号（一般就DMA、NORMAL）
7318         end_pfn = max(max_zone_pfn[i], start_pfn);
7319         arch_zone_lowest_possible_pfn[i] = start_pfn;
7320         arch_zone_highest_possible_pfn[i] = end_pfn;
7321 
7322         start_pfn = end_pfn;
7323     }
7324 
7325     /* Find the PFNs that ZONE_MOVABLE begins at in each node */
7326     memset(zone_movable_pfn, 0, sizeof(zone_movable_pfn));
    	 /*  用于计算进入ZONE_MOVABLE的内存数量  */
7327     find_zone_movable_pfns_for_nodes();
7328 
7329     /* 
			Print out the zone ranges 
			将各个内存域的最大、最小页帧号显示出来
		*/
7330     pr_info("Zone ranges:\n");
7331     for (i = 0; i < MAX_NR_ZONES; i++) {
7332         if (i == ZONE_MOVABLE)
7333             continue;
7334         pr_info("  %-8s ", zone_names[i]);
7335         if (arch_zone_lowest_possible_pfn[i] ==
7336                 arch_zone_highest_possible_pfn[i])
7337             pr_cont("empty\n");
7338         else
7339             pr_cont("[mem %#018Lx-%#018Lx]\n",
7340                 (u64)arch_zone_lowest_possible_pfn[i]
7341                     << PAGE_SHIFT,
7342                 ((u64)arch_zone_highest_possible_pfn[i]
7343                     << PAGE_SHIFT) - 1);
7344     }
7345 
7346     /* Print out the PFNs ZONE_MOVABLE begins at in each node */
7347     pr_info("Movable zone start for each node\n");
7348     for (i = 0; i < MAX_NUMNODES; i++) {
7349         if (zone_movable_pfn[i])
7350             pr_info("  Node %d: %#018Lx\n", i,
7351                    (u64)zone_movable_pfn[i] << PAGE_SHIFT);
7352     }
7353 
7354     /*
7355      * Print out the early node map, and initialize the
7356      * subsection-map relative to active online memory ranges to
7357      * enable future "sub-section" extensions of the memory map.
7358      */
7359     pr_info("Early memory node ranges\n");
7360     for_each_mem_pfn_range(i, MAX_NUMNODES, &start_pfn, &end_pfn, &nid) {
7361         pr_info("  node %3d: [mem %#018Lx-%#018Lx]\n", nid,
7362             (u64)start_pfn << PAGE_SHIFT,
7363             ((u64)end_pfn << PAGE_SHIFT) - 1);
7364         subsection_map_init(start_pfn, end_pfn - start_pfn);
7365     }
7366 
7367     /* Initialise every node */
7368     mminit_verify_pageflags_layout();
7369     setup_nr_node_ids();
7370     zero_resv_unavail();
    	 
    	 /*  代码遍历所有的活动结点，
          *  并分别对各个结点调用free_area_init_node建立数据结构，
          *  该函数需要结点第一个可用的页帧作为一个参数，
          *  而find_min_pfn_for_node则从early_node_map数组提取该信息   
          */
7371     for_each_online_node(nid) {
7372         pg_data_t *pgdat = NODE_DATA(nid);
    		 // 挨个内存节点初始化
7373         free_area_init_node(nid, NULL,
7374                 find_min_pfn_for_node(nid), NULL);
7375 
7376         /* Any memory on that node */
    		 /* 
              * 根据node_present_pages字段判断结点具有内存
              * 则在结点位图中设置N_HIGH_MEMORY标志
              * 该标志只表示结点上存在普通或高端内存
              * 因此check_for_regular_memory
              * 进一步检查低于ZONE_HIGHMEM的内存域中是否有内存
              * 并据此在结点位图中相应地设置N_NORMAL_MEMORY   
              */
7377         if (pgdat->node_present_pages)
7378             node_set_state(nid, N_MEMORY);
7379         check_for_memory(pgdat, nid);
7380     }
7381 }

/*
	功能：初始化所有结点的pg_data_t和zone、page的数据，并打印了管理区信息.
	file：mm/page_alloc.c
*/
6305 void __paginginit free_area_init_node(int nid, unsigned long *zones_size,
6306         unsigned long node_start_pfn, unsigned long *zholes_size)
6307 {
6308     pg_data_t *pgdat = NODE_DATA(nid);
6309     unsigned long start_pfn = 0;
6310     unsigned long end_pfn = 0;
6311 
6312     /* pg_data_t should be reset to zero when it's allocated */
6313     WARN_ON(pgdat->nr_zones || pgdat->kswapd_classzone_idx);
6314 
6315     pgdat->node_id = nid;
6316     pgdat->node_start_pfn = node_start_pfn;
6317     pgdat->per_cpu_nodestats = NULL;
6318 #ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
6319     get_pfn_range_for_nid(nid, &start_pfn, &end_pfn);
6320     pr_info("Initmem setup node %d [mem %#018Lx-%#018Lx]\n", nid,
6321         (u64)start_pfn << PAGE_SHIFT,
6322         end_pfn ? ((u64)end_pfn << PAGE_SHIFT) - 1 : 0);
6323 #else 
6324     start_pfn = node_start_pfn;
6325 #endif
    	 /*  首先累计各个内存域的页数
         *	计算结点中页的总数
         *	对连续内存模型而言
         *	这可以通过zone_sizes_init完成
         *	但calculate_node_totalpages还考虑了内存空洞 */
6326     calculate_node_totalpages(pgdat, start_pfn, end_pfn,
6327                   zones_size, zholes_size);
6328 
    	 /*  分配了该节点的页面描述符数组
     *  [pgdat->node_mem_map数组的内存分配  */
6329     alloc_node_mem_map(pgdat);
6330 #ifdef CONFIG_FLAT_NODE_MEM_MAP
6331     printk(KERN_DEBUG "free_area_init_node: node %d, pgdat %08lx, node_mem_map %08lx\n",
6332         nid, (unsigned long)pgdat,
6333         (unsigned long)pgdat->node_mem_map);
6334 #endif 
6335 
6336     reset_deferred_meminit(pgdat);
    	 /*  对该节点的每个区[DMA,NORMAL,HIGH]的的结构进行初始化  */
6337     free_area_init_core(pgdat);
6338 }

/*
	功能：初始化node的node_mem_map成员，及该node包含的页
	file：mm/page_alloc.c
	详解：
		1、全局的mem_map包含了所有的page
*/
6259 static void __ref alloc_node_mem_map(struct pglist_data *pgdat)
6260 {
6261     unsigned long __maybe_unused start = 0;
6262     unsigned long __maybe_unused offset = 0;
6263 
6264     /* Skip empty nodes */
6265     if (!pgdat->node_spanned_pages)
6266         return;
6267 
6268 #ifdef CONFIG_FLAT_NODE_MEM_MAP
6269     start = pgdat->node_start_pfn & ~(MAX_ORDER_NR_PAGES - 1);
6270     offset = pgdat->node_start_pfn - start;
6271     /* ia64 gets its own node_mem_map, before this, without bootmem */
6272     if (!pgdat->node_mem_map) {
6273         unsigned long size, end;
6274         struct page *map;
6275 
6276         /*
6277          * The zone's endpoints aren't required to be MAX_ORDER
6278          * aligned but the node_mem_map endpoints must be in order
6279          * for the buddy allocator to function correctly.
6280          */
6281         end = pgdat_end_pfn(pgdat);
6282         end = ALIGN(end, MAX_ORDER_NR_PAGES);
6283         size =  (end - start) * sizeof(struct page);
6284         map = alloc_remap(pgdat->node_id, size);
6285         if (!map)
6286             map = memblock_virt_alloc_node_nopanic(size,
6287                                    pgdat->node_id);
6288         pgdat->node_mem_map = map + offset;
6289     }
6290 #ifndef CONFIG_NEED_MULTIPLE_NODES
6291     /*
6292      * With no DISCONTIG, the global mem_map is just set as node 0's
6293      */
6294     if (pgdat == NODE_DATA(0)) {
6295         mem_map = NODE_DATA(0)->node_mem_map;
6296 #if defined(CONFIG_HAVE_MEMBLOCK_NODE_MAP) || defined(CONFIG_FLATMEM)
6297         if (page_to_pfn(mem_map) != pgdat->node_start_pfn)
6298             mem_map -= offset;
6299 #endif /* CONFIG_HAVE_MEMBLOCK_NODE_MAP */
6300     }
6301 #endif
6302 #endif /* CONFIG_FLAT_NODE_MEM_MAP */
6303 }

/*
	功能：
		1、初始化zone
		2、初始化page
	file：mm/page_alloc.c
*/
6159 /*
6160  * Set up the zone data structures:
6161  *   - mark all pages reserved
6162  *   - mark all memory queues empty
6163  *   - clear the memory bitmaps
6164  *  
6165  * NOTE: pgdat should get zeroed by caller.
6166  */
6167 static void __paginginit free_area_init_core(struct pglist_data *pgdat)
6168 {   
6169     enum zone_type j;
6170     int nid = pgdat->node_id;
6171  
         /*  初始化pgdat->node_size_lock自旋锁  */
6172     pgdat_resize_init(pgdat);
6173 #ifdef CONFIG_NUMA_BALANCING
6174     spin_lock_init(&pgdat->numabalancing_migrate_lock);
6175     pgdat->numabalancing_migrate_nr_pages = 0;
6176     pgdat->numabalancing_migrate_next_window = jiffies;
6177 #endif
6178 #ifdef CONFIG_TRANSPARENT_HUGEPAGE
6179     spin_lock_init(&pgdat->split_queue_lock);
6180     INIT_LIST_HEAD(&pgdat->split_queue);
6181     pgdat->split_queue_len = 0;
6182 #endif 
    	 /*  
    	 	初始化pgdat->kswapd_wait等待队列  
    	 	如果进程分配内存阻塞在等待kswapd进行内存回收，会挂载zone的等待队列上，当回收成功之后会唤醒
    	 	挂在等待队列上的进程
    	 */
6183     init_waitqueue_head(&pgdat->kswapd_wait);
    	 /*  初始化页换出守护进程创建空闲块的大小为2^kswapd_max_order  */
6184     init_waitqueue_head(&pgdat->pfmemalloc_wait);
6185 #ifdef CONFIG_COMPACTION
6186     init_waitqueue_head(&pgdat->kcompactd_wait);
6187 #endif
6188     pgdat_page_ext_init(pgdat);
6189     spin_lock_init(&pgdat->lru_lock);
6190     lruvec_init(node_lruvec(pgdat));
6191 
6192     pgdat->per_cpu_nodestats = &boot_nodestats;
6193 	 
         /* 遍历每个管理区 */
6194     for (j = 0; j < MAX_NR_ZONES; j++) {
6195         struct zone *zone = pgdat->node_zones + j;
6196         unsigned long size, realsize, freesize, memmap_pages;
6197         unsigned long zone_start_pfn = zone->zone_start_pfn;
6198 
             /*  size为该管理区中的页框数，包括洞 */
6199         size = zone->spanned_pages;
    		 /* realsize为管理区中的页框数，不包括洞 */
6200         realsize = freesize = zone->present_pages;
6201 
6202         /*
6203          * Adjust freesize so that it accounts for how much memory
6204          * is used by this zone for memmap. This affects the watermark
6205          * and per-cpu initialisations
6206          */
    		/* 调整realsize的大小，即减去page结构体占用的内存大小  */
        	/*  memmap_pags为包括洞的所有页框的page结构体所占的大小  */
6207         memmap_pages = calc_memmap_size(size, realsize);
6208         if (!is_highmem_idx(j)) {
6209             if (freesize >= memmap_pages) {
6210                 freesize -= memmap_pages;
6211                 if (memmap_pages)
6212                     printk(KERN_DEBUG
6213                            "  %s zone: %lu pages used for memmap\n",
6214                            zone_names[j], memmap_pages);
6215             } else /*  内存不够存放page结构体  */
6216                 pr_warn("  %s zone: %lu pages exceeds freesize %lu\n",
6217                     zone_names[j], memmap_pages, freesize);
6218         }
6219 
6220         /* Account for reserved pages */
    		 // 调整realsize的大小，即减去DMA保留页的大小
6221         if (j == 0 && freesize > dma_reserve) {
6222             freesize -= dma_reserve;
6223             printk(KERN_DEBUG "  %s zone: %lu pages reserved\n",
6224                     zone_names[0], dma_reserve);
6225         }
6226 
6227         if (!is_highmem_idx(j))
6228             nr_kernel_pages += freesize;
6229         /* Charge for highmem memmap if there are enough kernel pages */
6230         else if (nr_kernel_pages > memmap_pages * 2)
6231             nr_kernel_pages -= memmap_pages;
6232         nr_all_pages += freesize;
6233 
6234         /*
6235          * Set an approximate value for lowmem here, it will be adjusted
6236          * when the bootmem allocator frees pages into the buddy system.
6237          * And all highmem pages will be managed by the buddy system.
6238          */
    		 /* 设置zone->spanned_pages为包括洞的页框数  */
6239         zone->managed_pages = is_highmem_idx(j) ? realsize : freesize;
6240 #ifdef CONFIG_NUMA
    		 /* 设置zone所属的node号 */
6241         zone->node = nid;
6242 #endif
6243         zone->name = zone_names[j]; // 设置zone的名字
6244         zone->zone_pgdat = pgdat;	 // 设置zone所属的node的结构
6245         spin_lock_init(&zone->lock); /* 初始化各种锁 */
6246         zone_seqlock_init(zone);
6247         zone_pcp_init(zone); /* 尝试初始化该内存域的per-CPU缓存 */
6248 
6249         if (!size)
6250             continue;
6251 
6252         set_pageblock_order();
6253         setup_usemap(pgdat, zone, zone_start_pfn, size);
    		 /*
    		 	初始化zone的struct free_area    free_area[MAX_ORDER]成员，buddy从对应的zone中
    		 	分配page就是从free_area数组中获取合适的连续的物理page
    		 */
6254         init_currently_empty_zone(zone, zone_start_pfn, size);
    		 // 在for循环中初始化该zone管理的page
6255         memmap_init(size, nid, j, zone_start_pfn);
6256     }
6257 }

5512 #ifndef __HAVE_ARCH_MEMMAP_INIT
5513 #define memmap_init(size, nid, zone, start_pfn) \
5514     memmap_init_zone((size), (nid), (zone), (start_pfn), MEMMAP_EARLY)
5515 #endif
    
5414 /*
5415  * Initially all pages are reserved - free ones are freed
5416  * up by free_all_bootmem() once the early boot process is
5417  * done. Non-atomic initialization, single-pass.
5418  */
5419 void __meminit memmap_init_zone(unsigned long size, int nid, unsigned long zone,
5420         unsigned long start_pfn, enum memmap_context context)
5421 {
5422     struct vmem_altmap *altmap = to_vmem_altmap(__pfn_to_phys(start_pfn));
5423     unsigned long end_pfn = start_pfn + size;
5424     pg_data_t *pgdat = NODE_DATA(nid);
5425     unsigned long pfn;
5426     unsigned long nr_initialised = 0;
5427 #ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
5428     struct memblock_region *r = NULL, *tmp;
5429 #endif
5430 
5431     if (highest_memmap_pfn < end_pfn - 1)
5432         highest_memmap_pfn = end_pfn - 1;
5433 
5434     /*
5435      * Honor reservation requested by the driver for this ZONE_DEVICE
5436      * memory 
5437      */
5438     if (altmap && start_pfn == altmap->base_pfn)
5439         start_pfn += altmap->reserve;
5440 
5441     for (pfn = start_pfn; pfn < end_pfn; pfn++) {
5442         /*
5443          * There can be holes in boot-time mem_map[]s handed to this
5444          * function.  They do not exist on hotplugged memory.
5445          */
5446         if (context != MEMMAP_EARLY)
5447             goto not_early;
5448 
5449         if (!early_pfn_valid(pfn))
5450             continue;
5451         if (!early_pfn_in_nid(pfn, nid))
5452             continue;
5453         if (!update_defer_init(pgdat, pfn, end_pfn, &nr_initialised))
5454             break;
5455 
5456 #ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
5457         /*
5458          * Check given memblock attribute by firmware which can affect
5459          * kernel memory layout.  If zone==ZONE_MOVABLE but memory is
5460          * mirrored, it's an overlapped memmap init. skip it.
5461          */
5462         if (mirrored_kernelcore && zone == ZONE_MOVABLE) {
5463             if (!r || pfn >= memblock_region_memory_end_pfn(r)) {
5464                 for_each_memblock(memory, tmp)
5465                     if (pfn < memblock_region_memory_end_pfn(tmp))
5466                         break;
5467                 r = tmp;
5468             }
5469             if (pfn >= memblock_region_memory_base_pfn(r) &&
5470                 memblock_is_mirror(r)) {
5471                 /* already initialized as NORMAL */
5472                 pfn = memblock_region_memory_end_pfn(r);
5473                 continue;
5474             }
5475         }
5476 #endif
5477 
5478 not_early:
5479         /*
5480          * Mark the block movable so that blocks are reserved for
5481          * movable at startup. This will force kernel allocations
5482          * to reserve their blocks rather than leaking throughout
5483          * the address space during boot when many long-lived
5484          * kernel allocations are made.
5485          *
5486          * bitmap is created for zone's valid pfn range. but memmap
5487          * can be created for invalid pages (for alignment)
5488          * check here not to call set_pageblock_migratetype() against
5489          * pfn out of zone.
5490          */
5491         if (!(pfn & (pageblock_nr_pages - 1))) {
5492             struct page *page = pfn_to_page(pfn);
5493 
5494             __init_single_page(page, pfn, zone, nid);
5495             set_pageblock_migratetype(page, MIGRATE_MOVABLE);
5496             cond_resched();
5497         } else {
5498             __init_single_pfn(pfn, zone, nid);
5499         }
5500     }
5501 }
```

### 5、第四部创建了所有的pg_data_t节点，并且创建了每个节点的zone，为了更好的管理每个节点的所有zone，需要把每个节点的zone添加到pg_data_t的zonelist成员中

```c
/*
	file：mm/page_alloc.c
	功能：为所有的node添加zonelist
*/

5378 /*
5379  * unless system_state == SYSTEM_BOOTING.
5380  *
5381  * __ref due to call of __init annotated helper build_all_zonelists_init
5382  * [protected by SYSTEM_BOOTING].
5383  */     
5384 void __ref build_all_zonelists(pg_data_t *pgdat)
5385 {       
5386     if (system_state == SYSTEM_BOOTING) {
5387         build_all_zonelists_init();
5388     } else {
5389         __build_all_zonelists(pgdat);
5390         /* cpuset refresh routine should be here */
5391     }
5392     vm_total_pages = nr_free_pagecache_pages();
5393     /*
5394      * Disable grouping by mobility if the number of pages in the
5395      * system is too low to allow the mechanism to work. It would be
5396      * more accurate, but expensive to check per-zone. This check is
5397      * made on memory-hotadd so a system can start with mobility
5398      * disabled and enable it later
5399      */ 
5400     if (vm_total_pages < (pageblock_nr_pages * MIGRATE_TYPES))
5401         page_group_by_mobility_disabled = 1;
5402     else
5403         page_group_by_mobility_disabled = 0;
5404      
5405     pr_info("Built %i zonelists, mobility grouping %s.  Total pages: %ld\n",
5406         nr_online_nodes,
5407         page_group_by_mobility_disabled ? "off" : "on",
5408         vm_total_pages);
5409 #ifdef CONFIG_NUMA
5410     pr_info("Policy zone: %s\n", zone_names[policy_zone]);
5411 #endif
5412 }
```

### 6、节点、区域、page结构初始化完了，当kernel要使用内存的时候一般会通过buddy分配器分配，分配小于1个page的内存需要使用slub分配器，mm_init负责对应部分的初始化

```c
/*
	file：init/main.c
	功能：
		1、释放内存到buddy
		2、sla|u|ob初始化
*/ 
494  * Set up kernel memory allocators
 495  */
 496 static void __init mm_init(void)
 497 {  
 498     /*
 499      * page_ext requires contiguous pages,
 500      * bigger than MAX_ORDER unless SPARSEMEM.
 501      */
 502     page_ext_init_flatmem();
    	 // 释放内存到buddy
 503     mem_init();
    	 // sla|u|ob初始化
 504     kmem_cache_init();
 505     pgtable_init();
 506     vmalloc_init();
 507     ioremap_huge_init();
 508     /* Should be run before the first non-init thread is created */
 509     init_espfix_bsp();
 510     /* Should be run after espfix64 is set up. */
 511     pti_init();
 512 }
```

### 7、最后分配器初始化完成了，接着对slub的缓存初始化

```c
// 这个初始化与config选择的分配器（sla|u|ob）有关系

/*
	slab
*/
1319 void __init kmem_cache_init_late(void)
1320 {
1321     struct kmem_cache *cachep;
1322 
1323     slab_state = UP;
1324 
1325     /* 6) resize the head arrays to their final sizes */
1326     mutex_lock(&slab_mutex);
1327     list_for_each_entry(cachep, &slab_caches, list)
1328         if (enable_cpucache(cachep, GFP_NOWAIT))
1329             BUG();
1330     mutex_unlock(&slab_mutex);
1331     
1332     /* Done! */
1333     slab_state = FULL;
1334 
1335 #ifdef CONFIG_NUMA
1336     /*
1337      * Register a memory hotplug callback that initializes and frees
1338      * node.
1339      */
1340     hotplug_memory_notifier(slab_memory_callback, SLAB_CALLBACK_PRI);
1341 #endif
1342 
1343     /*
1344      * The reap timers are started later, with a module init call: That part
1345      * of the kernel is not yet operational.
1346      */
1347 }

/*
	slub
*/
4233 void __init kmem_cache_init_late(void)
4234 {
4235 }

/*
	slob
*/
658 void __init kmem_cache_init_late(void)
659 {
660     slab_state = FULL;
661 }
```



