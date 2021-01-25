### 1、概述

bio代表上层文件系统向通用块设备层发起的一次IO请求,request代表bio封装后的一次请求

### 2、结构体
```c
struct bio {
    struct bio      	*bi_next; // 连接下一个bio
    struct block_device *bi_bdev; // bio对应的block_device			
    unsigned int        bi_flags;  
    int         bi_error;
    unsigned long       bi_rw;
    struct bvec_iter    bi_iter;  
    unsigned int        bi_phys_segments;
    unsigned int        bi_seg_front_size;
    unsigned int        bi_seg_back_size; 
    atomic_t        __bi_remaining;
    bio_end_io_t        *bi_end_io;
    void            *bi_private;
#ifdef CONFIG_BLK_CGROUP
    struct io_context   *bi_ioc;
    struct cgroup_subsys_state *bi_css;
#endif
    union {
#if defined(CONFIG_BLK_DEV_INTEGRITY)
    struct bio_integrity_payload *bi_integrity; /* data integrity */
#endif
    };
    unsigned short      bi_vcnt;    /* how many bio_vec's */
    unsigned short      bi_max_vecs;    /* max bvl_vecs we can hold */
    atomic_t        __bi_cnt;   /* pin count */
    struct bio_vec      *bi_io_vec; /* the actual vec list */
    struct bio_set      *bi_pool;
    struct bio_vec      bi_inline_vecs[0];
};      
```

