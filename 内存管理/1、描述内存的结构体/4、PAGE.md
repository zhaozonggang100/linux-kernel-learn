### 1、内核数据结构  

include/linux/mm_types.h

```c
 31 /*
 32  * Each physical page in the system has a struct page associated with
 33  * it to keep track of whatever it is we are using the page for at the
 34  * moment. Note that we have no way to track which tasks are using
 35  * a page, though if it is a pagecache page, rmap structures can tell us
 36  * who is mapping it.
 37  *
 38  * The objects in struct page are organized in double word blocks in
 39  * order to allows us to use atomic double word operations on portions
 40  * of struct page. That is currently only used by slub but the arrangement
 41  * allows the use of atomic double word operations on the flags/mapping
 42  * and lru list pointers also.
 43  */
 44 struct page {
     	/*
     		页的标志，匿名页、file page等
     	*/
 45     /* First double word block */
 46     unsigned long flags;        /* Atomic flags, some possibly
 47                      * updated asynchronously */
 48     union {
     		/*
     			由于指针总长度是4的整数倍，mapping用低两位表示页映射标志
     			最低位PAGE_MAPPING_ANON表示匿名页
     			如果物理页是匿名页，page.mapping =（结构体anon_vma的地址 |PAGE_MAPPING_ANON）
     			如果物理页是文件页，page.mapping指向结构体address_space。
     		*/
 49         struct address_space *mapping;  /* If low bit clear, points to
 50                          * inode address_space, or NULL.
 51                          * If page mapped as anonymous
 52                          * memory, low bit is set, and
 53                          * it points to anon_vma object:
 54                          * see PAGE_MAPPING_ANON below.
 55                          */
 			/*
 				如果page是被slab分配器使用，s_mem指向slab中的第一个object
 			*/
 56         void *s_mem;            /* slab first object */
 57     };
 58 
 59     /* Second double word */
 60     struct {
 61         union {
     			/*
     				page在映射区里的偏移，单位是page
     				映射区：
     					1、如果是匿名映射，则代表在虚拟地址中的偏移
     					2、如果是文件映射，则代表在文件中的偏移
     			*/
 62             pgoff_t index;      /* Our offset within mapping. */
     
     			/*
     				如果page是slab分配器使用，代表slab缓存中空闲object的数量
     			*/
 63             void *freelist;     /* sl[aou]b first free object */
 64         };
  65 
 66         union {
 67 #if defined(CONFIG_HAVE_CMPXCHG_DOUBLE) && \
 68     defined(CONFIG_HAVE_ALIGNED_STRUCT_PAGE)
 69             /* Used for cmpxchg_double in slub */
 70             unsigned long counters;
 71 #else
 72             /*
 73              * Keep _count separate from slub cmpxchg_double data.
 74              * As the rest of the double word is protected by
 75              * slab_lock but _count is not.
 76              */
 77             unsigned counters;
 78 #endif
 79 
 80             struct {
 81 
 82                 union {
 83                     /*
 84                      * Count of ptes mapped in
 85                      * mms, to show when page is
 86                      * mapped & limit reverse map
 87                      * searches.
 88                      *
 89                      * Used also for tail pages
 90                      * refcounting instead of
 91                      * _count. Tail pages cannot
 92                      * be mapped and keeping the
 93                      * tail page _count zero at
 94                      * all times guarantees
 95                      * get_page_unless_zero() will
 96                      * never succeed on tail
 97                      * pages.
 98                      */
     					/*
     						该page被映射的次数，映射到多少个虚拟内存区域
     					*/
 99                     atomic_t _mapcount;
100 
101                     struct { /* SLUB */
102                         unsigned inuse:16;
103                         unsigned objects:15;
104                         unsigned frozen:1;
105                     };
106                     int units;  /* SLOB */
107                 };
108                 atomic_t _count;        /* Usage count, see below. */
109             };
110             unsigned int active;    /* SLAB */
111         };
112     };
113 
114     /*
115      * Third double word block
116      *
117      * WARNING: bit 0 of the first word encode PageTail(). That means
118      * the rest users of the storage space MUST NOT use the bit to
119      * avoid collision and false-positive PageTail().
120      */
121     union {
122         struct list_head lru;   /* Pageout list, eg. active_list
123                      * protected by zone->lru_lock !
124                      * Can be used as a generic list
125                      * by the page owner.
126                      */
127         struct {        /* slub per cpu partial pages */
128             struct page *next;  /* Next partial slab */
129 #ifdef CONFIG_64BIT
130             int pages;  /* Nr of partial slabs left */
131             int pobjects;   /* Approximate # of objects */
132 #else
133             short int pages;
134             short int pobjects;
135 #endif
136         };
137 
138         struct rcu_head rcu_head;   /* Used by SLAB
139                          * when destroying via RCU
140                          */
141         /* Tail pages of compound page */
142         struct {
143             unsigned long compound_head; /* If bit zero is set */
144 
145             /* First tail page only */
146 #ifdef CONFIG_64BIT
147             /*
148              * On 64 bit system we have enough space in struct page
149              * to encode compound_dtor and compound_order with
150              * unsigned int. It can help compiler generate better or
151              * smaller code on some archtectures.
152              */
153             unsigned int compound_dtor;
154             unsigned int compound_order;
155 #else
156             unsigned short int compound_dtor;
157             unsigned short int compound_order;
158 #endif
159         };
160 
161 #if defined(CONFIG_TRANSPARENT_HUGEPAGE) && USE_SPLIT_PMD_PTLOCKS
162         struct {
163             unsigned long __pad;    /* do not overlay pmd_huge_pte
164                          * with compound_head to avoid
165                          * possible bit 0 collision.
166                          */
167             pgtable_t pmd_huge_pte; /* protected by page->ptl */
168         };
169 #endif
170     };
171 
172     /* Remainder is not double word aligned */
173     union {
174         unsigned long private;      /* Mapping-private opaque data:
175                          * usually used for buffer_heads
176                          * if PagePrivate set; used for
177                          * swp_entry_t if PageSwapCache;
178                          * indicates order in the buddy
179                          * system if PG_buddy is set.
180                          */
181 #if USE_SPLIT_PTE_PTLOCKS
182 #if ALLOC_SPLIT_PTLOCKS
183         spinlock_t *ptl;
184 #else
185         spinlock_t ptl;
186 #endif
187 #endif
188         struct kmem_cache *slab_cache;  /* SL[AU]B: Pointer to slab */
189     };
190 
191 #ifdef CONFIG_MEMCG
192     struct mem_cgroup *mem_cgroup;
193 #endif
194 
195     /*
196      * On machines where all RAM is mapped into kernel address space,
197      * we can simply calculate the virtual address. On machines with
198      * highmem some memory is mapped into kernel virtual memory
199      * dynamically, so we need a place to store that address.
200      * Note that this field could be 16 bits on x86 ... ;)
201      *
202      * Architectures with slow multiplication can define
203      * WANT_PAGE_VIRTUAL in asm/page.h
204      */
205 #if defined(WANT_PAGE_VIRTUAL)
206     void *virtual;          /* Kernel virtual address (NULL if
207                        not kmapped, ie. highmem) */
208 #endif /* WANT_PAGE_VIRTUAL */
209 
210 #ifdef CONFIG_KMEMCHECK
211     /*
212      * kmemcheck wants to track the status of each byte in a page; this
213      * is a pointer to such a status block. NULL if not tracked.
214      */
215     void *shadow;
216 #endif
217 
218 #ifdef LAST_CPUPID_NOT_IN_PAGE_FLAGS
219     int _last_cpupid;
220 #endif
221 }
```

### 2、页用途分类  

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

### 3、页的结构定义  
1、一个页可能有多个用途，所以结构中用union表示了多个struct，但一个页同时只能处于一种用途  
2、页的种类  

```
1、文件读写页高速缓存或者匿名页（Page cache and anonymous pages）（可回收）
2、网络协议栈网卡DMA映射页（page_pool used by netstack）
3、slab、slub、slob页高速缓存
4、复合页（Tail pages of compound page）
```