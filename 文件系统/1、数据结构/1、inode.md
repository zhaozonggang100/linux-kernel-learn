### 1、inode概述

1、inode在Linux系统中代表一个真实存在的文件，块设备文件，文件系统中的文件等

2、根据文件所在的存储介质不同，磁盘设备中的文件inode存在于磁盘中，内存文件系统中的文件只存在于内存中。

3、Linux为了加速文件的读、写，引入了page cache和buffer cache，至于这两个结构读者可以自行了解，cache对应于inode的i_mapping成员（struct address_space），该成员使用内存page缓存了文件的内存，4.20以前是使用的`struct radix_tree_root  page_tree`（基树结构），4.20以后变成了xarray数组，详解查看`<https://blog.csdn.net/feelabclihu/article/details/105743389>`



### 2、结构体

1、inode结构体

```
struct inode {
	const struct inode_operations   *i_op;   // 对应的文件操作，读、写、创建等
    struct super_block  *i_sb;	// 文件所在的文件系统的超级快成员
	struct address_space    *i_mapping; // 文件的内存高速缓存
}
```

2、address_space结构就是正真的文件缓存

```
kernel：4.4.138
struct address_space {
    struct inode        *host;      /* owner: inode, block_device */
    struct radix_tree_root  page_tree;  /* radix tree of all pages */
    spinlock_t      tree_lock;  /* and lock protecting it */
    atomic_t        i_mmap_writable;/* count VM_SHARED mappings */
    struct rb_root      i_mmap;     /* tree of private and shared mappings */
    struct rw_semaphore i_mmap_rwsem;   /* protect tree, count, list */
    /* Protected by tree_lock together with the radix tree */
    unsigned long       nrpages;    /* number of total pages */
    unsigned long       nrshadows;  /* number of shadow entries */
    pgoff_t         writeback_index;/* writeback starts here */
    const struct address_space_operations *a_ops;   /* methods */
    unsigned long       flags;      /* error bits/gfp mask */
    spinlock_t      private_lock;   /* for use by the address_space */
    struct list_head    private_list;   /* ditto */
    void            *private_data;  /* ditto */
} __attribute__((aligned(sizeof(long))));

kernel：5.6
struct address_space {
    struct inode        *host;
    struct xarray       i_pages;	// 4.20内核之后使用xarray数组替换了基树
    gfp_t           gfp_mask;
    atomic_t        i_mmap_writable;
#ifdef CONFIG_READ_ONLY_THP_FOR_FS
    /* number of thp, only for non-shmem files */
    atomic_t        nr_thps;
#endif
    struct rb_root_cached   i_mmap;
    struct rw_semaphore i_mmap_rwsem;
    unsigned long       nrpages;
    unsigned long       nrexceptional;
    pgoff_t         writeback_index;
    const struct address_space_operations *a_ops;
    unsigned long       flags;
    errseq_t        wb_err;
    spinlock_t      private_lock;
    struct list_head    private_list;
    void            *private_data;
} __attribute__((aligned(sizeof(long)))) __randomize_layout;
```

