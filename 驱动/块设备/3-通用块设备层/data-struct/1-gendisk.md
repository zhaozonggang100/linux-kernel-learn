### 1、source

```c++
// include/linux/genhd.h
struct gendisk {
        /* major, first_minor and minors are input parameters only,
         * don't use directly.  Use disk_devt() and disk_max_parts().
         */
        int major;                      /* major number of driver */
        int first_minor;
        int minors;                     /* maximum number of minors, =1 for
                                         * disks that can't be partitioned. */

        char disk_name[DISK_NAME_LEN];  /* name of major driver */

        unsigned short events;          /* supported events */
        unsigned short event_flags;     /* flags related to event processing */

        /* Array of pointers to partitions indexed by partno.
         * Protected with matching bdev lock but stat and other
         * non-critical accesses use RCU.  Always access through
         * helpers.
         */
        // 指向磁盘的分区表
        struct disk_part_tbl __rcu *part_tbl;
        // 磁盘的0号分区，也是磁盘的第一个分区
        struct block_device *part0;

        const struct block_device_operations *fops;
        // 指向磁盘的请求队列
        struct request_queue *queue;
        // 通用磁盘对应的特定磁盘的私有数据，对于scsi为指向scsi_disk中的driver域；
        // 对于raid，为指向MD设备的指针
        // 对于device mapper，指向被映射设备
        void *private_data;

        int flags;
        unsigned long state;
#define GD_NEED_PART_SCAN               0
        // 指向sysfs/block/sdx这个磁盘下slaves目录对应的kobject指针
        struct kobject *slave_dir;

        struct timer_rand_state *random;
        atomic_t sync_io;               /* RAID */
        struct disk_events *ev;
#ifdef  CONFIG_BLK_DEV_INTEGRITY
        struct kobject integrity_kobj;
#endif  /* CONFIG_BLK_DEV_INTEGRITY */
#if IS_ENABLED(CONFIG_CDROM)
        struct cdrom_device_info *cdi;
#endif
        int node_id;
        struct badblocks *bb;
        struct lockdep_map lockdep_map;
}
```

### 2、info

linux内核将磁盘类设备的通用部分信息提取出来，表示为通用磁盘描述符**gendisk**，磁盘类驱动负责分配、初始化通用磁盘描述符，并将它添加到系统中。  

每个gendisk对应一个磁盘类设备，磁盘类设备描述符中定义一个指向gendisk描述符的指针（如disk域）