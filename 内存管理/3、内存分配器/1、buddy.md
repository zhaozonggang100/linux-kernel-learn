### 参考  
	https://www.cnblogs.com/tolimit/p/4610974.html  


​	
### 1、介绍  

buddy是以页为单位来管理物理内存的，提供了页面分配和释放的功能，buddy管理的物理页都包含在zone结构中，每一个node的所有zone形成链表。  
	
buddy系统将他所管理的物理页在zone中以2^order的方式存放在11（0-10）个链表中，最大的连续物理页是1024个连续页  
	
buddy是slab、vmalloc等实现的基础  

### 2、作用  

- 1、减少外部内存碎片  

	buddy系统在释放内存的时候，会尝试递归合并相邻的物理页，并插入高阶的链表中  
	
### 3、分配物理页  

- 1、api  

	alloc_pages

- 2、分配流程  

	1、UMA  
		alloc_pages-->alloc_pages_node--->__alloc_pages_node-->__alloc_pages--->__alloc_pages_nodemask  
	
	2、NUMA
		alloc_pages--->alloc_pages_current  

