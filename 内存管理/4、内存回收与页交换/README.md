### 参考  
```
https://blog.csdn.net/chenying126/article/details/79251843
```

### 1、介绍  

Linux为了在分配时为了解决内存不足，使用了两种机制来释放部分内存  
1、`不活跃`page cache的写回，然后直接释放对应的page cache页  
2、`不活跃`用户空间产生的匿名映射通过kswapd的交换机制交换到swap分区或者swapfile中  

当前内核回收策略已经从zone迁移到node了

### 2、从内存回收的角度页的分类  

- 1、能回收的页  

	用户进程产生的不活跃匿名页  
	用户进程文件读写产生不活跃的page cache  
	tmpfs产生的不活跃page cache  

- 2、不能回收的页  

	alloc_pages分配的页  
	用户调用mlock锁的页  

### 3、内存回收策略  
参考：https://blog.csdn.net/zqz_zqz/article/details/80333607  

- 1、kswapd进程周期性检测系统剩余空闲内存大小，和内存初始化时对应节点中zone中的_watermark（hgih、min、low）做对比，进行周期性回收内存  

	mm/vmscan.c中的kswapd()主逻辑  
	会通过zone中的水位标记进行回收，当空闲内存小于watermark[low]触发kswapd，直到剩余内存达到watermark[high]的时候停止。  

- 2、当分配内存大小大于剩余空闲内存时或者分配内存时小于watermark[min]值，强制触发内存回收  

	内存申请的时候进入slow path的内存申请逻辑进行回收  
	mm/page_alloc.c中的__alloc_pages_slowpath方法  

- 3、OOM

	当分配内存不足时，通过杀死占用内存最大的合适进程，释放掉进程对应的物理页  

### 4、内存回收流程  
`shrink_zones()--->shrink_lruvec()-->get_scan_count`

可能遍历到的LRU（最近）链表:
```
	LRU_INACTIVE_ANON：不活跃匿名页面链表
	LRU_ACTIVE_ANON：活跃匿名页面链表
	LRU_INACTIVE_FILE：不活跃文件映射页面链表
	LRU_ACTIVE_FILE：活跃文件映射页面链表	
	LRU_UNEVICTABLE：不可回收页面链表（不会被遍历）
```
每个zone都会包含上面五个链表，最新的版本，LRU链表已经转移到NODE结构中(pg_data_t)

### 5、proc
```
/proc/zoneinfo：查看水位标记  

/sys/module/lowmemorykiller/parameters/adj：0,100,200,300,900,906 （android给进程的被杀优先级数组） 
/sys/module/lowmemorykiller/parameters/minfree：18432,23040,27648,32256,55296,80640 （android给系统设置的内存阈值）

proc/sys/vm/swappiness
	100：匿名页面和page cache的LRU链表优先级相等  
    60：回收匿名页面多一些  
    0：4.0之后的内核只会回收page cache

    解释：
        值越高，内核就会越积极的使用swap；
        值越低，就会降低对swap的使用积极性。
        如果这个值为0，那么内存在free和file-backed使用的页面总量小于高水位标记
        （high water mark）之前，不会发生交换。
        
/proc/sys/vm/min_free_kbytes:内存最低水位
```

### 6、触发页回收的条件

1、申请分配页的时候，页分配器首先尝试使用低水线分配页。如果使用低水线分配失败，说明内存轻微不足，页分配器将会唤醒所有符合分配条件的内存节点的页回收线程，异步回收页，然后尝试使用最低水线分配页。如果分配失败，说明内存严重不足，页分配器将会直接回收页。如果直接回收页失败，那么判断是否应该重新尝试回收页。

2、kswpad线程周期性唤醒校测是否需要回收内存