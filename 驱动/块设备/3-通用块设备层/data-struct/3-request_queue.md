### 1、source

```c++
// include/linux/blkdev.h
struct request_queue {
        // 记录上次合并了bio的request，新的bio首先会尝试合并到这个request
        struct request          *last_merge;
        // 指向io调度器队列描述符的指针
        struct elevator_queue   *elevator;
        
        struct percpu_ref       q_usage_counter;

        struct blk_queue_stats  *stats;
        struct rq_qos           *rq_qos;

        const struct blk_mq_ops *mq_ops;
        
        /* sw queues */
        struct blk_mq_ctx __percpu      *queue_ctx;

        // 请求队列深度
        unsigned int            queue_depth;

        /* hw dispatch queues */
        struct blk_mq_hw_ctx    **queue_hw_ctx;
        unsigned int            nr_hw_queues;

        // 每个请求队列都都内嵌一个后备设备信息描述符，提供专属的请求处理机制（例如可以派生专门为它服务的flusher线程）
        struct backing_dev_info *backing_dev_info;
        
        /*
         * The queue owner gets to use this for whatever they like.
         * ll_rw_blk doesn't touch it.
         */
        // 请求队列所有者（块设备）的私有数据
        // 例如：
        //      scsi设备的请求队列将它指向scsi设备描述符，在为scsi设备分配请求队列时设置
        //      raid模块用来保存请求队列所属md设备的描述符
        void                    *queuedata;

        /*
         * various queue flags, see QUEUE_* below
         */
        unsigned long           queue_flags;
        /*
         * Number of contexts that have called blk_set_pm_only(). If this
         * counter is above zero then only RQF_PM requests are processed.
         */
        atomic_t                pm_only;

        /*
         * ida allocated id for this queue.  Used to index queues from
         * ioctx.
         */
        int                     id;

        /*
         * queue needs bounce pages for pages above this limit
         */
        // 用于反弹缓存区的分配标志。如果要请求的页面编号超过该值，则必须使用反弹缓存区
        gfp_t                   bounce_gfp;

        spinlock_t              queue_lock;

        /*
         * queue kobject
         */
        struct kobject kobj;

        /*
         * mq queue kobject
         */
        struct kobject *mq_kobj;

#ifdef  CONFIG_BLK_DEV_INTEGRITY
        struct blk_integrity integrity;
#endif  /* CONFIG_BLK_DEV_INTEGRITY */

#ifdef CONFIG_PM
        struct device           *dev;
        enum rpm_status         rpm_status;
        unsigned int            nr_pending;
#endif

        /*
         * queue settings
         */
        unsigned long           nr_requests;    /* Max # of requests */

        // 追加填充缓存区到一个request，使它对齐这个掩码。这将修改聚散列表的最会一项，让他包含填充缓存区
        unsigned int            dma_pad_mask;
        unsigned int            dma_alignment;

#ifdef CONFIG_BLK_INLINE_ENCRYPTION
        /* Inline crypto capabilities */
        struct blk_keyslot_manager *ksm;
#endif

        // 用于这个请求队列的请求超时时间
        unsigned int            rq_timeout;
        int                     poll_nsec;

        struct blk_stat_callback        *poll_cb;
        struct blk_rq_stat      poll_stat[BLK_MQ_POLL_STATS_BKTS];

        struct timer_list       timeout;
        struct work_struct      timeout_work;

        atomic_t                nr_active_requests_shared_sbitmap;

        struct list_head        icq_list;
#ifdef CONFIG_BLK_CGROUP
        DECLARE_BITMAP          (blkcg_pols, BLKCG_MAX_POLS);
        struct blkcg_gq         *root_blkg;
        struct list_head        blkg_list;
#endif

        // 请求队列的限制参数
        struct queue_limits     limits;

        unsigned int            required_elevator_features;

#ifdef CONFIG_BLK_DEV_ZONED
        /*
         * Zoned block device information for request dispatch control.
         * nr_zones is the total number of zones of the device. This is always
         * 0 for regular block devices. conv_zones_bitmap is a bitmap of nr_zones
         * bits which indicates if a zone is conventional (bit set) or
         * sequential (bit clear). seq_zones_wlock is a bitmap of nr_zones
         * bits which indicates if a zone is write locked, that is, if a write
         * request targeting the zone was dispatched. All three fields are
         * initialized by the low level device driver (e.g. scsi/sd.c).
         * Stacking drivers (device mappers) may or may not initialize
         * these fields.
         *
         * Reads of this information must be protected with blk_queue_enter() /
         * blk_queue_exit(). Modifying this information is only allowed while
         * no requests are being processed. See also blk_mq_freeze_queue() and
         * blk_mq_unfreeze_queue().
         */
        unsigned int            nr_zones;
        unsigned long           *conv_zones_bitmap;
        unsigned long           *seq_zones_wlock;
        unsigned int            max_open_zones;
        unsigned int            max_active_zones;
#endif /* CONFIG_BLK_DEV_ZONED */

        /*
         * sg stuff
         */
        unsigned int            sg_timeout;
        unsigned int            sg_reserved_size;
        int                     node;
        struct mutex            debugfs_mutex;
#ifdef CONFIG_BLK_DEV_IO_TRACE
        struct blk_trace __rcu  *blk_trace;
#endif
        /*
         * for flush operations
         */
        struct blk_flush_queue  *fq;

        struct list_head        requeue_list;
        spinlock_t              requeue_lock;
        struct delayed_work     requeue_work;

        struct mutex            sysfs_lock;
        struct mutex            sysfs_dir_lock;

        /*
         * for reusing dead hctx instance in case of updating
         * nr_hw_queues
         */
        struct list_head        unused_hctx_list;
        spinlock_t              unused_hctx_lock;

        int                     mq_freeze_depth;

#if defined(CONFIG_BLK_DEV_BSG)
        struct bsg_class_device bsg_dev;
#endif

#ifdef CONFIG_BLK_DEV_THROTTLING
        /* Throttle data */
        struct throtl_data *td;
#endif
        struct rcu_head         rcu_head;
        wait_queue_head_t       mq_freeze_wq;
        /*
         * Protect concurrent access to q_usage_counter by
         * percpu_ref_kill() and percpu_ref_reinit().
         */
        struct mutex            mq_freeze_lock;

                struct blk_mq_tag_set   *tag_set;
        struct list_head        tag_set_list;
        struct bio_set          bio_split;

        struct dentry           *debugfs_dir;

#ifdef CONFIG_BLK_DEBUG_FS
        struct dentry           *sched_debugfs_dir;
        struct dentry           *rqos_debugfs_dir;
#endif

        bool                    mq_sysfs_init_done;

        size_t                  cmd_size;

#define BLK_MAX_WRITE_HINTS     5
        u64                     write_hints[BLK_MAX_WRITE_HINTS];
};
```

### 2、info

- 1、一个磁盘（gendisk）对应一个request_queue

- 2、sysfs

```bash
root@PC-15068:/sys/block/sdb/queue# ls -l
total 0
-rw-r--r-- 1 root root 4096 3月  12 15:02 add_random
-r--r--r-- 1 root root 4096 3月  12 15:02 chunk_sectors
-r--r--r-- 1 root root 4096 3月  12 15:02 dax
-r--r--r-- 1 root root 4096 3月  12 15:02 discard_granularity
-rw-r--r-- 1 root root 4096 3月  12 15:02 discard_max_bytes
-r--r--r-- 1 root root 4096 3月  12 15:02 discard_max_hw_bytes
-r--r--r-- 1 root root 4096 3月  12 15:02 discard_zeroes_data
-r--r--r-- 1 root root 4096 3月  12 15:02 fua
-r--r--r-- 1 root root 4096 3月  12 15:02 hw_sector_size
-rw-r--r-- 1 root root 4096 3月  12 15:02 io_poll
-rw-r--r-- 1 root root 4096 3月  12 15:02 io_poll_delay
drwxr-xr-x 2 root root    0 3月  11 14:18 iosched
-rw-r--r-- 1 root root 4096 3月  12 15:02 iostats
-rw-r--r-- 1 root root 4096 3月  12 15:02 io_timeout
-r--r--r-- 1 root root 4096 3月  12 15:02 logical_block_size
-r--r--r-- 1 root root 4096 3月  12 15:02 max_discard_segments
-r--r--r-- 1 root root 4096 3月  12 15:02 max_hw_sectors_kb
-r--r--r-- 1 root root 4096 3月  12 15:02 max_integrity_segments
-rw-r--r-- 1 root root 4096 3月  12 15:02 max_sectors_kb
-r--r--r-- 1 root root 4096 3月  12 15:02 max_segments
-r--r--r-- 1 root root 4096 3月  12 15:02 max_segment_size
-r--r--r-- 1 root root 4096 3月  12 15:02 minimum_io_size
-rw-r--r-- 1 root root 4096 3月  12 15:02 nomerges
-rw-r--r-- 1 root root 4096 3月  12 15:02 nr_requests                   # 请求队列最多能缓存的请求数量
-r--r--r-- 1 root root 4096 3月  12 15:02 nr_zones
-r--r--r-- 1 root root 4096 3月  12 15:02 optimal_io_size
-r--r--r-- 1 root root 4096 3月  12 15:02 physical_block_size
-rw-r--r-- 1 root root 4096 3月  12 15:02 read_ahead_kb
-rw-r--r-- 1 root root 4096 3月  11 10:24 rotational
-rw-r--r-- 1 root root 4096 3月  12 15:02 rq_affinity
-rw-r--r-- 1 root root 4096 3月  12 15:02 scheduler
-rw-r--r-- 1 root root 4096 3月  12 15:02 wbt_lat_usec
-rw-r--r-- 1 root root 4096 3月  12 15:02 write_cache
-r--r--r-- 1 root root 4096 3月  12 15:02 write_same_max_bytes
-r--r--r-- 1 root root 4096 3月  12 15:02 write_zeroes_max_bytes
-r--r--r-- 1 root root 4096 3月  12 15:02 zone_append_max_bytes
-r--r--r-- 1 root root 4096 3月  12 15:02 zoned
```