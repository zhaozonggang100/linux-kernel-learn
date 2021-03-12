### 1、source

```c++
// include/linux/blkdev.h
/*
 * Try to put the fields that are referenced together in the same cacheline.
 *
 * If you modify this structure, make sure to update blk_rq_init() and
 * especially blk_mq_rq_ctx_init() to take care of the added fields.
 */
struct request {
        // 指向包含这个请求的请求队列
        struct request_queue *q;
        struct blk_mq_ctx *mq_ctx;
        struct blk_mq_hw_ctx *mq_hctx;

        unsigned int cmd_flags;         /* op and common flags */
        req_flags_t rq_flags;

        int tag;
        int internal_tag;

        /* the following two fields are internal, NEVER access directly */
        unsigned int __data_len;        /* total data len：请求的长度，单位为字节 */
        sector_t __sector;              /* sector cursor ：请求的起始扇区编号*/

        struct bio *bio;                /* 还没有完成传输的第一个bio */
        struct bio *biotail;            /* 最后一个bio */

        struct list_head queuelist;     /* 用于将该请求链入请求派发队列或者IO调度队列的"连接件" */

        /*
         * The hash is used inside the scheduler, and killed once the
         * request reaches the dispatch list. The ipi_list is only used
         * to queue the request for softirq completion, which is long
         * after the request has been unhashed (and even removed from
         * the dispatch list).
         */
        union {
                // 链入电梯算法哈希表的的连接件，按照最后一个扇区的编号进行hash，以方便查找可以合并的对象
                struct hlist_node hash; /* merge hash */
                struct list_head ipi_list;
        };

        /*
         * The rb_node is only used inside the io scheduler, requests
         * are pruned when moved to the dispatch queue. So let the
         * completion_data share space with the rb_node.
         */
        union {
                // rb_node只在调度器内部使用，completion_data只在请求处理完之后使用
                struct rb_node rb_node; /* sort/lookup */       /* 链入电梯算法红黑树的连接件，以方便排序和查找 */
                struct bio_vec special_vec;
                void *completion_data;                          /* 请求完成处理时指向已完成的命令（如：scsi_command） */
                int error_count; /* for legacy drivers, don't use */
        };

        /*
         * Three pointers are available for the IO schedulers, if they need
         * more they have to dynamically allocate it.  Flush requests are
         * never put on the IO scheduler. So let the flush fields share
         * space with the elevator data.
         */
        union {
                struct {
                        struct io_cq            *icq;
                        void                    *priv[2];
                } elv;

                struct {
                        unsigned int            seq;
                        struct list_head        list;
                        rq_end_io_fn            *saved_end_io;
                } flush;
        };

        struct gendisk *rq_disk;        /* 请求所对应的通用磁盘描述符 */
        struct block_device *part;      /* 请求所对应的分区 */
#ifdef CONFIG_BLK_RQ_ALLOC_TIME
        /* Time that the first bio started allocating this request. */
        u64 alloc_time_ns;
#endif
        /* Time that this request was allocated for this IO. */
        u64 start_time_ns;              /* 请求开始创建的时间 */
        /* Time that I/O was submitted to the device. */
        u64 io_start_time_ns;

#ifdef CONFIG_BLK_WBT
        unsigned short wbt_flags;
#endif
        /*
         * rq sectors used for blk stats. It has the same value
         * with blk_rq_sectors(rq), except that it never be zeroed
         * by completion.
         */
        unsigned short stats_sectors;

        /*
         * Number of scatter-gather DMA addr+len pairs after
         * physical address coalescing is performed.
         */
        unsigned short nr_phys_segments;        /* 请求在物理内存中占据的不连续的段数目，sg列表的数目 */

#if defined(CONFIG_BLK_DEV_INTEGRITY)
        unsigned short nr_integrity_segments;
#endif

#ifdef CONFIG_BLK_INLINE_ENCRYPTION
        struct bio_crypt_ctx *crypt_ctx;
        struct blk_ksm_keyslot *crypt_keyslot;
#endif

        unsigned short write_hint;
        unsigned short ioprio;          /* 请求的优先级 */

        enum mq_rq_state state;
        refcount_t ref;                 /* 请求的引用计数 */

        unsigned int timeout;           /* 该请求的请求超时时间 */
        unsigned long deadline;         /* 请求必须完成的时间，为开始执行的时间加上超时时间 */

        union {
                struct __call_single_data csd;
                u64 fifo_time;
        };

        /*
         * completion callback.
         */
        rq_end_io_fn *end_io;           /* 请求完成的回调函数。例如在scsi命令处理完成后被调用进行后续处理 */
        void *end_io_data;              /* 配合上述完成回调函数的数据，例如指向一个complete变量以等待scsi命令执行完成 */
};
```

### 2、info

- 1、IO调度算法负责将请求从IO调度队列转移至派发队列

- 2、每个块设备层的请求表示一段**连续的**扇区访问