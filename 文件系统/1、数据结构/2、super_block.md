### 1、概要  

- 1、超级块对象存在于两个链表中  

&emsp;&emsp;&emsp;&emsp; 1、所有的超级块对象被链接到一个循环双向链表中，链表的第一个元素由**super_blocks**变量表示，而超级块结构的s_list域保存指向链表中相邻元素的指针。  

&emsp;&emsp;&emsp;&emsp; 2、此外，每个超级块还被链接到它所属的文件系统类型的超级块实例链表中，这个链表的连接件是**s_instances**

- 2、文件系统支持的文件的最大size在alloc_super（fs/super.c）分配超级块时设置的默认值是(2^32-1)，某些文件系统有特殊需求，应该重新设置这个值。

- 3、基于FUSE（用户空间文件系统）的文件系统，文件系统的类型表示有些麻烦。从内核的角度看，只有两种文件系统：`fuse`和`fuseblk`。但是从用户的角度，可以有多种不同的文件系统类型。用户甚至不关心这个文件系统是不是基于FUSE。因此，基于FUSE的文件系统会用到子类型（**s_subtype**）,在fstab、mtab和/proc/mounts等中显示为`type.subtype`的形式。

### 2、data struct

```c++
// file ：include/linux/fs.h
struct super_block {
        // 链入到所有超级块对象链表的"连接件"
        struct list_head        s_list;         /* Keep this first */
        // 存储超级块信息的块设备的设备号（主设备号：此设备号）
        dev_t                   s_dev;          /* search index; _not_ kdev_t */
        // 文件系统的块长度的位数
        unsigned char           s_blocksize_bits;
        // 文件系统的块长度（单位：字节）
        unsigned long           s_blocksize;
        // 文件的最大长度
        loff_t                  s_maxbytes;     /* Max file size */
        // 指向具体的文件系统，ext4指向fs/ext4/super.c中定义的ext4_fs_type
        struct file_system_type *s_type;
        // 指向具体文件系统，ext4指向fs/ext4/super.c中定义的ext4_sops
        const struct super_operations   *s_op;
        // 指向磁盘配额操作表的指针，exxt4指向fs/ext4/super.c中定义的ext4_quota_operations
        const struct dquot_operations   *dq_op;
        // 指向磁盘配额管理操作表的指针，ext4指向fs/ext4/super.c中定义的ext4_qctl_operations
        const struct quotactl_ops       *s_qcop;
        // 指向导出操作表的指针，ext4指向fs/ext4/super.c中定义的ext4_export_ops
        const struct export_operations *s_export_op;
        // 装载文件系统标识
        unsigned long           s_flags;
        unsigned long           s_iflags;       /* internal SB_I_* flags */
        // 文件系统魔术，ext4为0xEF53，不同种类文件系统都不一样
        unsigned long           s_magic;
        // 指向文件系统根目录dentry对象
        struct dentry           *s_root;
        // 用于卸载文件系统的信号量
        struct rw_semaphore     s_umount;
        // 引用计数？
        int                     s_count;
        /*
                活动引用计数，super_block中的引用计数有两个，一个是s_count，另一个是s_active，s_count是真正的引用计数，表示这个super_block是否能被释放，s_active表示被mount的次数，不管s_active值为多少，反映到s_count中，一律算做S_BIAS个s_count
        */
        atomic_t                s_active;
#ifdef CONFIG_SECURITY
        // 指向superblock安全结构的指针
        void                    *s_security;
#endif
        // 指向超级块扩展属性结构的指针
        // ext指向：fs/ext4/xattr.c中定义的const struct xattr_handler *ext4_xattr_handlers[]指针数组
        const struct xattr_handler **s_xattr;

        const struct fscrypt_operations *s_cop;

        // 文件系统的匿名dentry哈希表表头，用于处理网络文件系统
        struct hlist_bl_head    s_anon;         /* anonymous dentries for (nfs) exporting */
        struct list_head        s_mounts;       /* list of mounts; _not_ for fs use */
        // 对于磁盘文件系统为指向块设备描述符的指针，否则为NULL
        struct block_device     *s_bdev;
        /*
                指向后备设备信息描述符的指针，对于某些磁盘文件系统，指向块设备请求队列的内嵌后备设备信息；某些网络文件系统会定义自己的后备设备信息，其他文件系统是空操作
        */
        struct backing_dev_info *s_bdi;
        // 对于基于MTD的超级块，至下关MTD信息结构的指针
        struct mtd_info         *s_mtd;
        // 链接到所属文件系统类型的超级块实例链表的"连接件"
        struct hlist_node       s_instances;
        // 支持的磁盘配额类型位图
        unsigned int            s_quota_types;  /* Bitmask of supported quota types */
        // 磁盘配额信息描述符
        struct quota_info       s_dquot;        /* Diskquota specific options */

        struct sb_writers       s_writers;

        // 对于磁盘文件系统为块设备名，否则为文件系统名
        char s_id[32];                          /* Informational name */
        // 具体文件系统超级块的UUID，相同文件系统的超级块的UUID不同
        u8 s_uuid[16];                          /* UUID */
        // 指向具体文件系统超级块信息的指针（内存中的超级块）,每个文件系统的私有私有结构
        // ext4指向fs/ext4/ext4.h中定义的ext4_sb_info
        // 其中包含了磁盘分区位掩码和不被纳入VFS公共模型的其他数据
        void                    *s_fs_info;     /* Filesystem private info */
        unsigned int            s_max_links;
        // 对于磁盘文件系统，记录装载模式（只读、或读/写）
        fmode_t                 s_mode;

        /* Granularity of c/m/atime in ns.
           Cannot be worse than a second */
        // 文件系统的时间戳（访问/修改时间），单位是纳秒
        u32                s_time_gran;

        /*
         * The next field is for VFS *only*. No filesystems have any business
         * even looking at it. You had been warned.
         */
        // 跨目录重命名文件时使用的信号量，只限VFS内部使用
        struct mutex s_vfs_rename_mutex;        /* Kludge */

        /*
         * Filesystem subtype.  If non-empty the filesystem type field
         * in /proc/mounts will be "type.subtype"
         */
        // 文件系统的子类型
        char *s_subtype;

        /*
         * Saved mount options for lazy filesystems using
         * generic_show_options()
         */
        // 保存装载选项，以便以后显示，配合generic_show_options()使用
        char __rcu *s_options;
        const struct dentry_operations *s_d_op; /* default d_op for dentries */

        /*
         * Saved pool identifier for cleancache (-1 means none)
         */
        int cleancache_poolid;

        struct shrinker s_shrink;       /* per-sb shrinker handle */

        /* Number of inodes with nlink == 0 but still referenced */
        atomic_long_t s_remove_count;

        /* Being remounted read-only */
        int s_readonly_remount;

        /* AIO completions deferred from interrupt context */
        struct workqueue_struct *s_dio_done_wq;
        struct hlist_head s_pins;

        /*
         * Keep the lru lists last in the structure so they always sit on their
         * own individual cachelines.
         */
        struct list_lru         s_dentry_lru ____cacheline_aligned_in_smp;
        struct list_lru         s_inode_lru ____cacheline_aligned_in_smp;
        struct rcu_head         rcu;
        struct work_struct      destroy_work;

        struct mutex            s_sync_lock;    /* sync serialisation lock */

        /*
         * Indicates how deep in a filesystem stack this SB is
         */
        int s_stack_depth;

        /* s_inode_list_lock protects s_inodes */
        spinlock_t              s_inode_list_lock ____cacheline_aligned_in_smp;
        /* 
                1、文件系统所有inode链表的头
                2、每个inode通过inode的i_sb_list链入这个链表        
        */
        struct list_head        s_inodes;       /* all inodes */
};


// file：include/linux/fs.h
struct super_operations {
        struct inode *(*alloc_inode)(struct super_block *sb);
        void (*destroy_inode)(struct inode *);
        // 日志文件系统用来将inode标记为脏
        void (*dirty_inode) (struct inode *, int flags);
        // VFS核心需要将inode信息写到磁盘上时调用，内部只是将内存中的inode标记脏，然后一次性通过blockdev mapping将所有这些inode写回到磁盘
        int (*write_inode) (struct inode *, struct writeback_control *wbc);
        // inode的最后一个引用被释放时调用
        int (*drop_inode) (struct inode *);
        void (*evict_inode) (struct inode *);
        // VFS核心释放超级块时调用
        void (*put_super) (struct super_block *);
        // VFS核心写出和一个超级块关联的所有"脏数据"时被调用
        int (*sync_fs)(struct super_block *sb, int wait);
        int (*freeze_super) (struct super_block *);
        // 锁住文件系统，强制使它进入一致性，用于LVM
        int (*freeze_fs) (struct super_block *);
        int (*thaw_super) (struct super_block *);
        int (*unfreeze_fs) (struct super_block *);
        // eg：dumpe2fs|tun2fs获取ext4超级块信息时调用
        int (*statfs) (struct dentry *, struct kstatfs *);
        // 文件系统重新装载时调用
        int (*remount_fs) (struct super_block *, int *, char *);
        int (*remount_fs2) (struct vfsmount *, struct super_block *, int *, char *);
        void *(*clone_mnt_data) (void *);
        void (*copy_mnt_data) (void *, void *);
        // vfs卸载文件系统时调用
        void (*umount_begin) (struct super_block *);

        int (*show_options)(struct seq_file *, struct dentry *);
        int (*show_options2)(struct vfsmount *,struct seq_file *, struct dentry *);
        int (*show_devname)(struct seq_file *, struct dentry *);
        int (*show_path)(struct seq_file *, struct dentry *);
        int (*show_stats)(struct seq_file *, struct dentry *);
#ifdef CONFIG_QUOTA
        ssize_t (*quota_read)(struct super_block *, int, char *, size_t, loff_t);
        ssize_t (*quota_write)(struct super_block *, int, const char *, size_t, loff_t);
        struct dquot **(*get_dquots)(struct inode *);
#endif
        int (*bdev_try_to_free_page)(struct super_block*, struct page*, gfp_t);
        long (*nr_cached_objects)(struct super_block *,
                                  struct shrink_control *);
        long (*free_cached_objects)(struct super_block *,
                                    struct shrink_control *);
};
```