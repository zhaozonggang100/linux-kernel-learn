### 1、内核数据结构  

	`struct page`

### 2、成员  
1、unsigned long flags 	//原子标志，可能会被异步更新  

`该页属于的节点，属于节点中哪个zone，页框属性（读、写、属于user或者kernel等）`  

2、struct list_head lru  

`该页属于zone中buddy或者每cpu高速缓存中哪个链表`  

3、atomic_t _refcount  

`页的引用计数`

### 3、页用途分类  

1、不可移动的页  

`内核中使用的页大部分属于这种页`  

2、可回收页（不可移动）  
```
page cache
	匿名页映射：用户堆、栈、数据段、代码端
tmpfs使用的页
	inode、dentry使用的slab高速缓存
```

3、可移动页

### 4、页的结构定义  
1、一个页可能有多个用途，所以结构中用union表示了多个struct，但一个页同时只能处于一种用途  
2、页的种类  
```
1、文件读写页高速缓存或者匿名页（Page cache and anonymous pages）（可回收）
2、网络协议栈网卡DMA映射页（page_pool used by netstack）
3、slab、slub、slob页高速缓存
4、复合页（Tail pages of compound page）
```