### 1、slab  

- 1、介绍  

	slab是基于buddy的对象（object）高速缓存。  
	每个slab高速缓存都属于一个kmem_cache结构管理  
	每个kmem_cache中都包含三个对象链表（kmem_list3）：  
		1、占用满对象链表  
		2、未满对象链表  
		3、已满对象链表  
	cache_chain会将内核中所有的kmem_cache组织在一个双向链表中  

- 2、作用  

	1、分配小于一页的内存，减少buddy只能分配最小以页为单位的内存，从而减少buddy的内部碎片问题  
	
	2、提供给内核经常使用的对象一个对象池，释放对象的时候不立刻释放对象占用的内存  
	kmalloc  
	inode  
	dentry  
	skb_buff  
	task_struct  
	file_struct	  
	

3、释放不用slab对象  
	1、根据配置的参数，在kmem_cache_free释放一个对象时发现超过缓存池的极限值就会释放不用的slab中的对象  
	2、slab分配器有一个定时器，扫描不用的slab释放  

### 2、slub  


### 3、slob  