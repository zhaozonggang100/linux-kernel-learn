### 1、概要

&emsp;&emsp; 1、struct inode是指所有文件系统共有内存VFS管理的inode结构,如ext4中的inode定义为`struct ext4_inode_info`，bdevfs的inode结构为`struct bdev_inode`.

&emsp;&emsp; 2、除根inode外，所有的inode加入到一个全局哈希表inode_hashtable中，其哈希项的索引计算基于所属文件系统的超级块描述符的地址以及inode编号。这个哈希表的作用是方便查找属于某个文件系统的特定的inode。

&emsp;&emsp; 3、每个inode还被包含在所属文件系统的superblock以s_inodes域为首的双向循环链表中，inode的i_sb_list域存放了该链表中的相连元素的指针。

&emsp;&emsp; 4、inode可以表示多种对象，包括目录、文件、符号链接、字符设备、块设备、sock设备、管道设备

### 2、data struct  bdev_inode

```c++
// file : include/linux/fs.h
/*
 * Keep mostly read-only and often accessed (especially for
 * the RCU path lookup and 'stat' data) fields at the beginning
 * of the 'struct inode'
 */
struct inode {
        // 文件类型和访问权限
        umode_t                 i_mode;
        unsigned short          i_opflags;
        // 创建该文件的用户id
        kuid_t                  i_uid;
        // 创建该文件的用户组id
        kgid_t                  i_gid;
        // 文件系统装载标志
        unsigned int            i_flags;

#ifdef CONFIG_FS_POSIX_ACL
        struct posix_acl        *i_acl;
        struct posix_acl        *i_default_acl;
#endif
        // 操作inode表的函数集指针
        const struct inode_operations   *i_op;
        // 指向所属的sb的指针
        struct super_block      *i_sb;
        // 该文件对应的内存地址空间，文件读写的时候会有cache/buffer缓存，指向i_data域
        struct address_space    *i_mapping;

#ifdef CONFIG_SECURITY
        void                    *i_security;
#endif

        /* Stat data, not accessed from path walking */
        // inode编号
        unsigned long           i_ino;
        /*
         * Filesystems may only read i_nlink directly.  They shall use the
         * following functions for modification:
         *
         *    (set|clear|inc|drop)_nlink
         *    inode_(inc|dec)_link_count
         */
        union {
                // inode的硬链接数
                const unsigned int i_nlink;
                unsigned int __i_nlink;
        };
        // 如果本inode代表一个块设备，那么该域代表块设备的设备号
        dev_t                   i_rdev;
        // 以字节为单位表示的文件长度
        loff_t                  i_size;
        // 文件的最后访问时间
        struct timespec64       i_atime;
        // 文件的最后修改时间
        struct timespec64       i_mtime;
        // inode的最后修改时间
        struct timespec64       i_ctime;
        spinlock_t              i_lock; /* i_blocks, i_bytes, maybe i_size */
                unsigned short          i_bytes;
        // 文件块长度的位数
        u8                      i_blkbits;
        u8                      i_write_hint;
        /*
         * 文件系统数据管理的入口：代表文件使用的块数
         *  1、以前的方式是长度为32的数组
         *  2、现在是一个64位的整数，包含了ext4_extent_header和ext4_extent_idx，在该
         *  数据结构中，只有叶子节点中存储的数据包含文件逻辑地址与磁盘物理地址的映射关
         * 系。在数据管理中有3个关键的数据结构，分别是ext4_extent_header、
         * ext4_extent_idx和ext4_extent。
         */
        blkcnt_t                i_blocks;

#ifdef __NEED_I_SIZE_ORDERED
        seqcount_t              i_size_seqcount;
#endif

        /* Misc */ // inode状态标记
        unsigned long           i_state;
        struct rw_semaphore     i_rwsem;

        /*
                这个文件关联的地址空间的page第一次被"脏"的时间，被writeback代码确定是否将这个inode写回磁盘
        */
        unsigned long           dirtied_when;   /* jiffies of first dirtying */
        unsigned long           dirtied_time_when;

        struct hlist_node       i_hash;
        struct list_head        i_io_list;      /* backing dev IO list */
#ifdef CONFIG_CGROUP_WRITEBACK
        struct bdi_writeback    *i_wb;          /* the associated cgroup wb */

        /* foreign inode detection, see wbc_detach_inode() */
        int                     i_wb_frn_winner;
        u16                     i_wb_frn_avg_time;
        u16                     i_wb_frn_history;
#endif
        struct list_head        i_lru;          /* inode LRU list */
        // 链接到所属文件系统超级块的inode链表的"连接件"
        struct list_head        i_sb_list;
        struct list_head        i_wb_list;      /* backing dev writeback list */
        union {
                // 引用这个inode的dentry链表的表头
                struct hlist_head       i_dentry;
                // 访问dentry链表的rcu锁
                struct rcu_head         i_rcu;
        };
        // 版本号，在每次使用后递增
        atomic64_t              i_version;
        atomic64_t              i_sequence; /* see futex */
        // inode的引用计数器
        atomic_t                i_count;
        atomic_t                i_dio_count;
        // 写inode的进程计数器
        atomic_t                i_writecount;
#if defined(CONFIG_IMA) || defined(CONFIG_FILE_LOCKING)
        atomic_t                i_readcount; /* struct files open RO */
#endif
        union {
                // 指向文件操作表的指针
                const struct file_operations    *i_fop; /* former ->i_op->default_file_ops */
                void (*free_inode)(struct inode *);
        };
        struct file_lock_context        *i_flctx;
        // 文件的地址空间对象
        struct address_space    i_data;
        /*
                如果inode代表一个块设备，则该域链入到块设备的slave inode链表（表头为block_device结构的bd_inodes域）的连接件。如果代表一个字符设备，则该域链入到字符设备的inode链表（表头为cdev结构的list域）的连接件.
        */
        struct list_head        i_devices;
        union {
                struct pipe_inode_info  *i_pipe;
                struct cdev             *i_cdev;
                char                    *i_link;
                unsigned                i_dir_seq;
        };
        // inode版本号，在某些文件系统中使用
        __u32                   i_generation;

#ifdef CONFIG_FSNOTIFY
        // 这个inode关心的所有事件
        __u32                   i_fsnotify_mask; /* all events this inode cares about */
        struct fsnotify_mark_connector __rcu    *i_fsnotify_marks;
#endif

#ifdef CONFIG_FS_ENCRYPTION
        struct fscrypt_info     *i_crypt_info;
#endif

#ifdef CONFIG_FS_VERITY
        struct fsverity_info    *i_verity_info;
#endif
        // 文件系统或设备驱动的私有指针
        void                    *i_private; /* fs or device private pointer */
} __randomize_layout;


/*
        file : include/linux/fs.h
*/
struct inode_operations {
        struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
        const char * (*get_link) (struct dentry *, struct inode *, struct delayed_call *);
        int (*permission) (struct inode *, int);
        struct posix_acl * (*get_acl)(struct inode *, int);

        int (*readlink) (struct dentry *, char __user *,int);

        // 只对目录有效，用来在目录下创建常规文件
        int (*create) (struct inode *,struct dentry *, umode_t, bool);
        int (*link) (struct dentry *,struct inode *,struct dentry *);
        int (*unlink) (struct inode *,struct dentry *);
        int (*symlink) (struct inode *,struct dentry *,const char *);
        // 在目录inode下创建目录
        int (*mkdir) (struct inode *,struct dentry *,umode_t);
        int (*rmdir) (struct inode *,struct dentry *);
        // 在目录inode下创建块设备文件或字符设备文件
        int (*mknod) (struct inode *,struct dentry *,umode_t,dev_t);
        int (*rename) (struct inode *, struct dentry *,
                        struct inode *, struct dentry *, unsigned int);
        int (*setattr) (struct dentry *, struct iattr *);
        int (*getattr) (const struct path *, struct kstat *, u32, unsigned int);
        ssize_t (*listxattr) (struct dentry *, char *, size_t);
        int (*fiemap)(struct inode *, struct fiemap_extent_info *, u64 start,
                      u64 len);
        int (*update_time)(struct inode *, struct timespec64 *, int);
        int (*atomic_open)(struct inode *, struct dentry *,
                           struct file *, unsigned open_flag,
                           umode_t create_mode);
        int (*tmpfile) (struct inode *, struct dentry *, umode_t);
        int (*set_acl)(struct inode *, struct posix_acl *, int);
} ____cacheline_aligned;


// file : include/linux/fs.h
/**
 * struct address_space - Contents of a cacheable, mappable object.
 * @host: Owner, either the inode or the block_device.
 * @i_pages: Cached pages.
 * @gfp_mask: Memory allocation flags to use for allocating pages.
 * @i_mmap_writable: Number of VM_SHARED mappings.
 * @nr_thps: Number of THPs in the pagecache (non-shmem only).
 * @i_mmap: Tree of private and shared mappings.
 * @i_mmap_rwsem: Protects @i_mmap and @i_mmap_writable.
 * @nrpages: Number of page entries, protected by the i_pages lock.
 * @nrexceptional: Shadow or DAX entries, protected by the i_pages lock.
 * @writeback_index: Writeback starts here.
 * @a_ops: Methods.
 * @flags: Error bits and flags (AS_*).
 * @wb_err: The most recent error which has occurred.
 * @private_lock: For use by the owner of the address_space.
 * @private_list: For use by the owner of the address_space.
 * @private_data: For use by the owner of the address_space.
 */
struct address_space {
        struct inode            *host;
        struct xarray           i_pages;
        gfp_t                   gfp_mask;
        atomic_t                i_mmap_writable;
#ifdef CONFIG_READ_ONLY_THP_FOR_FS
        /* number of thp, only for non-shmem files */
        atomic_t                nr_thps;
#endif
        struct rb_root_cached   i_mmap;
        struct rw_semaphore     i_mmap_rwsem;
        unsigned long           nrpages;
        unsigned long           nrexceptional;
        pgoff_t                 writeback_index;
        const struct address_space_operations *a_ops;
        unsigned long           flags;
        errseq_t                wb_err;
        spinlock_t              private_lock;
        struct list_head        private_list;
        void                    *private_data;
} __attribute__((aligned(sizeof(long)))) __randomize_layout;

// file : include/linux/fs.h
struct file_operations {
        struct module *owner;
        loff_t (*llseek) (struct file *, loff_t, int);
        ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
        ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
        ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
        ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
        int (*iopoll)(struct kiocb *kiocb, bool spin);
        int (*iterate) (struct file *, struct dir_context *);
        int (*iterate_shared) (struct file *, struct dir_context *);
        // 进程检测文件是否有活动，被select和poll使用
        __poll_t (*poll) (struct file *, struct poll_table_struct *);
        long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
        long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
        // mmap系统调用使用
        int (*mmap) (struct file *, struct vm_area_struct *);
        unsigned long mmap_supported_flags;
        // vfs打开inode时调用，创建一个file描述符，为后面读写文件做准备
        int (*open) (struct inode *, struct file *);
        int (*flush) (struct file *, fl_owner_t id);
        // 打开文件的最后一个引用被关闭时使用
        int (*release) (struct inode *, struct file *);
        int (*fsync) (struct file *, loff_t, loff_t, int datasync);
        int (*fasync) (int, struct file *, int);
        int (*lock) (struct file *, int, struct file_lock *);
        ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
        unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
        int (*check_flags)(int);
        int (*flock) (struct file *, int, struct file_lock *);
        ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
        ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
        int (*setlease)(struct file *, long, struct file_lock **, void **);
        long (*fallocate)(struct file *file, int mode, loff_t offset,
                          loff_t len);
        void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
        unsigned (*mmap_capabilities)(struct file *);
#endif
        ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
                        loff_t, size_t, unsigned int);
        loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
                                   struct file *file_out, loff_t pos_out,
                                   loff_t len, unsigned int remap_flags);
        int (*fadvise)(struct file *, loff_t, loff_t, int);
} __randomize_layout;

struct file {
        union {
                struct llist_node       fu_llist;
                struct rcu_head         fu_rcuhead;
        } f_u;
        struct path             f_path;
        struct inode            *f_inode;       /* cached value */
        const struct file_operations    *f_op;

        /*
         * Protects f_ep, f_flags.
         * Must not be taken from IRQ context.
         */
        spinlock_t              f_lock;
        enum rw_hint            f_write_hint;
        atomic_long_t           f_count;
        unsigned int            f_flags;
        fmode_t                 f_mode;
        struct mutex            f_pos_lock;
        loff_t                  f_pos;
        struct fown_struct      f_owner;
        const struct cred       *f_cred;
        struct file_ra_state    f_ra;

        u64                     f_version;
#ifdef CONFIG_SECURITY
        void                    *f_security;
#endif
        /* needed for tty driver, and maybe others */
        void                    *private_data;

#ifdef CONFIG_EPOLL
        /* Used by fs/eventpoll.c to link all the hooks to this file */
        struct hlist_head       *f_ep;
#endif /* #ifdef CONFIG_EPOLL */
        struct address_space    *f_mapping;
        errseq_t                f_wb_err;
        errseq_t                f_sb_err; /* for syncfs */
} __randomize_layout
  __attribute__((aligned(4)));  /* lest something weird decides that 2 is OK */
```