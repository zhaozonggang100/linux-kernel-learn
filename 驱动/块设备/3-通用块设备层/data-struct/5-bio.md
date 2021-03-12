### 1、source

```c++
// include/linux/blk_types.h
/*
 * main unit of I/O for the block layer and lower layers (ie drivers and
 * stacking drivers)
 */
struct bio {
        // 用来将bio链入request包含的bio链表
        struct bio              *bi_next;       /* request queue link */
        // bio对应的通用磁盘描述符
        struct gendisk          *bi_disk;
        unsigned int            bi_opf;         /* bottom bits req flags,
                                                 * top bits REQ_OP. Use
                                                 * accessors.
                                                 */
        unsigned short          bi_flags;       /* status, etc and bvec pool number */
        unsigned short          bi_ioprio;
        unsigned short          bi_write_hint;
        blk_status_t            bi_status;
        u8                      bi_partno;              /* bio对应的磁盘分区编号 */
        atomic_t                __bi_remaining;

        struct bvec_iter        bi_iter;

        bio_end_io_t            *bi_end_io;             /* 在bio完成IO操作的回调函数 */

        void                    *bi_private;            /* 被通用块层和块设备驱动的IO完成方法使用，如果需要将bio分裂，分裂后的bio通过该域指向 */
#ifdef CONFIG_BLK_CGROUP
        /*
         * Represents the association of the css and request_queue for the bio.
         * If a bio goes direct to device, it will not have a blkg as it will
         * not have a request_queue associated with it.  The reference is put
         * on release of the bio.
         */
        struct blkcg_gq         *bi_blkg;
        struct bio_issue        bi_issue;
#ifdef CONFIG_BLK_CGROUP_IOCOST
        u64                     bi_iocost_cost;
#endif
#endif

#ifdef CONFIG_BLK_INLINE_ENCRYPTION
        struct bio_crypt_ctx    *bi_crypt_context;
#endif

        union {
#if defined(CONFIG_BLK_DEV_INTEGRITY)
                struct bio_integrity_payload *bi_integrity; /* data integrity */
#endif
        };

        // 这个bio的bio_vec数组包含的segment的数量
        unsigned short          bi_vcnt;        /* how many bio_vec's */

        /*
         * Everything starting with bi_max_vecs will be preserved by bio_reset()
         */
        // bio的bio_vec数组的最大项数
        unsigned short          bi_max_vecs;    /* max bvl_vecs we can hold */

        atomic_t                __bi_cnt;       /* pin count */

        // bio的bio_vec数组
        struct bio_vec          *bi_io_vec;     /* the actual vec list */

        struct bio_set          *bi_pool;

        /*
         * We can inline a number of vecs at the end of the bio, to avoid
         * double allocations for a small number of bio_vecs. This member
         * MUST obviously be kept at the very end of the bio.
         */
        struct bio_vec          bi_inline_vecs[];
};

// include/linux/bvec.h
/**
 * struct bio_vec - a contiguous range of physical memory addresses
 * @bv_page:   First page associated with the address range.
 * @bv_len:    Number of bytes in the address range.
 * @bv_offset: Start of the address range relative to the start of @bv_page.
 *
 * The following holds for a bvec if n * PAGE_SIZE < bv_offset + bv_len:
 *
 *   nth_page(@bv_page, n) == @bv_page + n
 *
 * This holds because page_is_mergeable() checks the above property.
 */
struct bio_vec {
        struct page     *bv_page;
        unsigned int    bv_len;
        unsigned int    bv_offset;
};
```

### 2、info

- 1、bio的数据在物理扇区上是连续的

- 2、多个扇区相邻的bio可能会合并成一个块设备驱动的request

- 3、bio的多个segment（bio_vec）是逐个被复制到块设备驱动（scsi|mmc|...）的sg列表