### 1、source

```c++
// include/linux/blk_types.h
struct block_device {
        // 分区在磁盘内的起始扇区编号
        sector_t                bd_start_sect;
        // 磁盘统计信息，包括以处理的读/写I/O数、已读/写的I/O扇区数等
        struct disk_stats __percpu *bd_stats;
        // 用于修正（round off）分区统计信息的时间
        unsigned long           bd_stamp;
        // 分区是否为只读，1：只读，0：不是只读
        bool                    bd_read_only;   /* read-only policy */
        // 分区对应的dev下面的块设备文件设备号
        dev_t                   bd_dev;
        // 打开该块设备的次数
        int                     bd_openers;
        // 这个分区块设备在bdev文件系统中的inode指针
        struct inode *          bd_inode;       /* will die */
        // 格式化分区之后指向分区对应的文件系统内存中的超级块
        struct super_block *    bd_super;
        struct mutex            bd_mutex;       /* open/close mutex */
        void *                  bd_claiming;
        // 内嵌的设备模型
        struct device           bd_device;
        // 分区如果被上层dm设备映射，分区的（/sys/block/sdx）holders目录中包含被哪些逻辑设备映射
        void *                  bd_holder;
        int                     bd_holders;
        bool                    bd_write_holder;
#ifdef CONFIG_SYSFS
        // 在sysfs块设备的目录建立holders目录
        struct list_head        bd_holder_disks;
#endif
        struct kobject          *bd_holder_dir;
        // 分区号
        u8                      bd_partno;
        /* number of times partitions within this device have been opened. */
        unsigned                bd_part_count;
        
        spinlock_t              bd_size_lock; /* for bd_inode->i_size updates */
        // 指向分区对应的磁盘的gendisk结构
        struct gendisk *        bd_disk;
        struct backing_dev_info *bd_bdi;

        /* The counter of freeze processes */
        int                     bd_fsfreeze_count;
        /* Mutex for freeze */
        struct mutex            bd_fsfreeze_mutex;
        struct super_block      *bd_fsfreeze_sb;

        struct partition_meta_info *bd_meta_info;
#ifdef CONFIG_FAIL_MAKE_REQUEST
        bool                    bd_make_it_fail;
#endif
} __randomize_layout;

// include/linux/genhd.h
// gendisk【struct disk_part_tbl __rcu *part_tbl;】分区表
// 磁盘的分区数量是由限制的，动态进行扩展
struct disk_part_tbl {
        struct rcu_head rcu_head;
        // 分区个数
        int len;
        // 最后一次查找分区的指针
        struct block_device __rcu *last_lookup;
        // 分区数组
        struct block_device __rcu *part[];
};
```

### 2、info

- 1、2.6内核用`hd_struct`代表一个分区，5.10内核（可能更早之前）用`block_device`代表一个分区