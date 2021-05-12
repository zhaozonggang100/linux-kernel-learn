### ref

```
https://blog.csdn.net/zwj0403/article/details/38409061
https://blog.csdn.net/ooonebook/article/details/55272522
https://blog.csdn.net/feiying0canglang/article/details/101778289#读写流程
https://blog.csdn.net/weixin_38006908/article/details/87375404?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase
https://blog.csdn.net/shenjin_s/article/details/80855762?utm_source=blogxgwz2
https://www.cnblogs.com/linhaostudy/p/10818152.html [介绍高通sdhci-msm mmc host framework]
https://www.cnblogs.com/linhaostudy/p/10813200.html

```

### 1、CARD
- 1、包含了mmc_driver也就是mmc card的驱动（所有符合MMC协议规范的card通用）  
```
drivers/mmc/card/block.c
drivers/mmc/card/queue.c
```

- 2、涉及到的结构体  

1、struct gendisk 代表一个块设备或者分区，是块设备IO子系统中对接文件系统层的，文件系统生成bio会在通用块设备层进入gendisk的request队列  
```
struct gendisk {
    int major; // 磁盘或者分区的主设备号
    int first_minor; // 磁盘的第一个次设备号
    int minors; // 磁盘包含的分区数
    char disk_name[DISK_NAME_LEN]; // 磁盘或者分区名
    struct request_queue *queue; // 请求队列
    const struct block_device_operations *fops; // 操作函数集
}
```

2、struct request_queue，用来和gendisk绑定，作为gendisk的请求队列  
```
struct request_queue {
    struct list_head    queue_head;
    struct request      *last_merge;
    struct elevator_queue   *elevator; // 电梯队列
    make_request_fn     *make_request_fn;
    softirq_done_fn     *softirq_done_fn;
    struct blk_mq_ops   *mq_ops;
    struct bio_set      *bio_split; // bio集合
}
```

3、struct mmc_blk_data， 每个slot一个mmc_blk_data，绑定一个分区  
```
struct mmc_blk_data {	
            struct mmc_queue queue; // 块设备或者分区对应的mmc封装的请求队列
            struct gendisk *disk;  // 代表一个块设备或者分区
            unsigned int    usage; // 是否在使用
            unsigned int    read_only; // 只读属性
            unsigned int    part_type;
            unsigned int    name_idx; 
            unsigned int    reset_done;
            unsigned int    part_curr;  // 分区号
            int area_type;  // 分区类型，boot、rpmb、gp、main
}
```

4、struct mmc_queue，mmc的card驱动对应请求队列，内部包含struct request_queue  
```
struct mmc_queue {
    struct mmc_card     *card;
    // mmc绑定task从IO调度器队列fetch request
    struct task_struct  *thread;  
    // 从IO调度器fetch的request交给该函数处理（这里是初始化为mmc_blk_issue_rq）
    int (*issue_fn)(struct mmc_queue *, struct request *); 
    // data会在mmc_blk_issue_rq处理request时赋值给mmc_blk_data结构
    void            *data;
    // 通用块设备的请求队列
    struct request_queue    *queue;
    struct mmc_queue_req    mqrq[2];
    struct mmc_queue_req    *mqrq_cur; // mqrq[0]
    struct mmc_queue_req    *mqrq_prev; // mqrq[1]
}
```

5、mmc驱动对request的封装

```
struct mmc_queue_req {
    struct request      *req;			// 来自gendisk的request
    struct mmc_blk_request  brq;
    struct scatterlist  *sg;
    char            *bounce_buf;
    struct scatterlist  *bounce_sg;
    unsigned int        bounce_sg_len;
    struct mmc_async_req    mmc_active;
    enum mmc_packed_type    cmd_type;
    struct mmc_packed   *packed;
};
struct mmc_blk_request {
    struct mmc_request  mrq;
    struct mmc_command  sbc;
    struct mmc_command  cmd;
    struct mmc_command  stop;
    struct mmc_data     data;
    int    retune_retry_done;
};
struct scatterlist {
#ifdef CONFIG_DEBUG_SG
    unsigned long   sg_magic;
#endif
    unsigned long   page_link;
    unsigned int    offset;
    unsigned int    length;
    dma_addr_t  dma_address;
#ifdef CONFIG_NEED_SG_DMA_LENGTH
    unsigned int    dma_length;
#endif
};
struct mmc_async_req {
    /* active mmc request */
    struct mmc_request  *mrq;
    /*  
     * Check error status of completed mmc request.
     * Returns 0 if success otherwise non zero.
     */
    int (*err_check) (struct mmc_card *, struct mmc_async_req *); 
};
```

6、struct mmc_request ，交给core处理的request

```
struct mmc_request {
    struct mmc_command  *sbc;       /* SET_BLOCK_COUNT for multiblock */
    struct mmc_command  *cmd;
    struct mmc_data     *data;
    struct mmc_command  *stop;

    struct completion   completion;
    void         (*done)(struct mmc_request *); /* completion function */
    struct mmc_host     *host;
};
```

7、struct mmc_card ，代表一个符合MMC标准的card(EMMC、SD、SDIO)

```
struct mmc_card {
    struct mmc_host     *host; // card连接的host
    struct device       dev; // 设备模型device
    unsigned int        rca; // card对应的EMMC中rca寄存器中emmc的地址
    unsigned int        type; // 这里就是EMMC CARD
    unsigned int        quirks; // card的癖好
    unsigned int        erase_size; // 一次性擦除的块大小
    unsigned int        erase_shift;
    // debugfs的目录，一般/sys/kernel/debug/mmc
    struct dentry       *debugfs_root; 
    struct mmc_part part[MMC_NUM_PHY_PARTITION]; // 分区
    unsigned int    nr_parts; // 分区数
}
```

8、struct mmc_data；包含真正要处理的数据  

```
struct mmc_data {
    unsigned int        timeout_ns; /* data timeout (in ns, max 80ms) */
    unsigned int        timeout_clks;   /* data timeout (in clocks) */
    unsigned int        blksz;      /* data block size */
    unsigned int        blocks;     /* number of blocks */
    int         error;      		/* data error */
    unsigned int        flags;
	unsigned int        bytes_xfered;
    struct mmc_command  *stop;      /* stop command */
    struct mmc_request  *mrq;       /* associated request */
    unsigned int        sg_len;     /* size of scatter list */
    int         sg_count;   		/* mapped sg entries */
    struct scatterlist  *sg;        /* DMA处理的数据,是一个scatterlist的内存										散列表 */
    s32         host_cookie;    	/* host private data */
};
```

9、在内核空间和用户空间需要交互大量数据用到scatterlist（散列表），物理内存散列表，以页为单位管理
```
https://www.cnblogs.com/alantu2018/p/8457417.html
struct scatterlist {
    unsigned long   page_link;	// 指示该内存块所在的页面
    unsigned int    offset;		// 指示该内存块在页面中的偏移
    unsigned int    length;		// 该内存块的长度
    dma_addr_t  dma_address;	// 该内存块实际的起始地址
#ifdef CONFIG_NEED_SG_DMA_LENGTH    
    unsigned int    dma_length;	// 相应的长度信息
#endif
};
```

10、host用来和emmc直接物理通信
```
struct mmc_host {
	struct device       *parent;
	struct device       class_dev;	// sys/class/mmc_host/包含了所有的host
	const struct mmc_host_ops *ops; // host操作函数集，发送命令等
	struct mmc_ios      ios; // mmc总线的 io setting
	struct dentry       *debugfs_root; // debugfs调试
	struct mmc_async_req    *areq; // 已经激活的异步请求
	struct mmc_context_info context_info; // 异步请求的信息
	struct mmc_card     *card; // host attach的card
	wait_queue_head_t   wq; // 发给host的请求被挂起的进程列表，当能处理时唤醒
	struct mmc_slot     slot;
	const struct mmc_bus_ops *bus_ops;
	// 非常重要的成员，不同的host的区别都在这里
	unsigned long       private[0] ____cacheline_aligned;
}
struct mmc_host_ops {
	// 完成一次请求，可能用到DMA
	void (*post_req)(struct mmc_host *host, struct mmc_request *req,
    			int err);
	// 准备一次请求
	void (*pre_req)(struct mmc_host *host, struct mmc_request *req,
               bool is_first_req);
   	// 真正的请求处理函数，sdhci标准的实现函数是sdhci_request
    void    (*request)(struct mmc_host *host, struct mmc_request *req);
    // io setting
    void    (*set_ios)(struct mmc_host *host, struct mmc_ios *ios);
}
```

11、厂商的host，符合sdhci标准的host  
```
struct sdhci_host {
	const char *hw_name;	// mmc  bus  name
	int irq; // 厂商host的中断
	void __iomem *ioaddr; // 设备寄存器的内存映射
	struct mmc_host *mmc; // 所属的抽象的mmc_host
	struct mmc_host_ops mmc_host_ops;
	u64 dma_mask;
	u64 coherent_dma_mask;
	unsigned int version;
	struct mmc_request *mrq; // 当前的request
	struct mmc_command *cmd; // 当前的命令
	struct mmc_data *data; // 当前的data
	unsigned long private[0] ____cacheline_aligned; // 厂商定义的host
}
https://www.lagou.com/lgeduarticle/12542.html
struct sdhci_msm_host {
	struct mmc_host  *mmc;
}
```

- 3、 初始化流程

参考：https://blog.csdn.net/dengziliang001/article/details/52042837?locationNum=1&fps=1

1、 注册MMC card驱动初始化入口  
```
// 注册mmc_driver到mmc core，mmc bus会进行card和mmc_driver的match
static struct mmc_driver mmc_driver = {
	.probe	= mmc_blk_probe;
};
int mmc_blk_init(void)
{
	res = mmc_register_driver(&mmc_driver);  
}
module_init(mmc_blk_init); 
```

2、当mmc core的 mmc bus检测到mmc_driver注册之后，会和core事先注册的card进行匹配，从而调用  

```
int mmc_blk_probe(struct mmc_card *card)
	struct mmc_blk_data *md, *part_md;  // 每个slot一个mmc_blk_data  
	md = mmc_blk_alloc(card);	// 分配并返回mmc_blk_data内部包含gendisk
	if (IS_ERR(md))
        return PTR_ERR(md);

    string_get_size((u64)get_capacity(md->disk), 512, STRING_UNITS_2,
            cap_str, sizeof(cap_str));
    pr_info("%s: %s %s %s %s\n",
        md->disk->disk_name, mmc_card_id(card), mmc_card_name(card),
        cap_str, md->read_only ? "(ro)" : "");

    if (mmc_blk_alloc_parts(card, md))
        goto out;

	// 将上面申请的md赋值给card对应的struct device dev成员的data成员
    dev_set_drvdata(&card->dev, md);

#ifdef CONFIG_MMC_BLOCK_DEFERRED_RESUME
    mmc_set_bus_resume_policy(card->host, 1);
#endif
	
	// 内部调用内核提供的add_disk将上面分配的gendisk（md->disk）添加到系统，标识该磁盘可以启用
    if (mmc_add_disk(md))
        goto out;

    list_for_each_entry(part_md, &md->part, part) {
        if (mmc_add_disk(part_md))
            goto out;
    }

    pm_runtime_use_autosuspend(&card->dev);
    pm_runtime_set_autosuspend_delay(&card->dev, MMC_AUTOSUSPEND_DELAY_MS);
}
```

3、mmc_blk_alloc调用mmc_blk_alloc_req执行真正的分配  
```
struct mmc_blk_data *mmc_blk_alloc_req(struct mmc_card *card,
    struct device *parent,sector_t size,bool default_ro,
    const char *subname, int area_type) 
{
    struct mmc_blk_data *md;  // 包含gendisk的md，代表mmc类磁盘描述符
    md = kzalloc(sizeof(struct mmc_blk_data), GFP_KERNEL);
    md->read_only = mmc_blk_readonly(card); // 设置card只读
    md->disk = alloc_disk(perdev_minors); // 分配gendisk
    // 初始化mmc_blk_data->mmc_queue(queue)，mmc_queue中包含request_queue
    ret = mmc_init_queue(&md->queue, card, NULL, subname, area_type); 
    
    /*下面设置mmc磁盘类描述符mmc_blk_data的mmc_queue请求处理函数和私有域*/
    // mmc_blk_issue_rq函数用来处理从IO调度层fetch的request
    md->queue.issue_fn = mmc_blk_issue_rq;
    // 设置mmc_blk_data->mmc_queue->data(void*)
    md->queue.data = md;
    
    /*下面设置mmc磁盘类描述符的gendisk对象的成员*/
    // 设置mmc_blk_data->gendisk->major，主设备号179
    md->disk->major = MMC_BLOCK_MAJOR;
    // 设置第一个次设备号
    md->disk->first_minor = devidx * perdev_minors;
    // 对gendisk的操作函数函数集
    md->disk->fops = &mmc_bdops;
    // gendisk的私有域private_data指向mmc_blk_data
    md->disk->private_data = md;
    // gendisk的请求队列和mmc_blk_queue->mmc_queue->request_queue都指向IO子系统的请求队列
    md->disk->queue = md->queue.queue;
    // 设置disk只读
    set_disk_ro(md->disk, md->read_only || default_ro);
}
```

4、mmc_init_queue会初始化mmc_queue对应的内核线程等  
```
int mmc_init_queue(struct mmc_queue *mq, struct mmc_card *card,
			spinlock_t *lock, const char *subname, int area_type)
{
    struct mmc_host *host = card->host;
    struct mmc_queue_req *mqrq_cur = &mq->mqrq[0];
    struct mmc_queue_req *mqrq_prev = &mq->mqrq[1];
    mq->card = card;

	// mmc request的处理函数
    mq->queue = blk_init_queue(mmc_request_fn, lock);
    
    // mmc_prep_request过滤不合法request
    blk_queue_prep_rq(mq->queue, mmc_prep_request);
    
    // 当文件系统来的请求通过mmc_request_fn处理后会判断条件唤醒内核线程
    mq->thread = kthread_run(mmc_queue_thread, mq, "mmcqd/%d%s",
    host->index, subname ? subname : "");
}

void mmc_request_fn(struct request_queue *q)
{
	struct mmc_queue *mq = q->queuedata;
    struct request *req;
    unsigned long flags;
    struct mmc_context_info *cntx;

    if (!mq) {
        while ((req = blk_fetch_request(q)) != NULL) {
            req->cmd_flags |= REQ_QUIET;
            __blk_end_request_all(req, -EIO);
        }
        return;
    }   

    cntx = &mq->card->host->context_info;
    if (!mq->mqrq_cur->req && mq->mqrq_prev->req) {
        /*
         * New MMC request arrived when MMC thread may be
         * blocked on the previous request to be complete
         * with no current request fetched
         */
        spin_lock_irqsave(&cntx->lock, flags);
        if (cntx->is_waiting_last_req) {
            cntx->is_new_req = true;
            wake_up_interruptible(&cntx->wait);
        }
        spin_unlock_irqrestore(&cntx->lock, flags);
    } else if (!mq->mqrq_cur->req && !mq->mqrq_prev->req)
    	// 唤醒内核线程mmc_queue_thread处理请求
        wake_up_process(mq->thread);
}
```

5、mmc_queue_thread内核线程在不处理来自通用块设备层的request时会进入TASK_INTERRUPTIBLE状态  

```
int mmc_queue_thread(void *d) 
{   // d就是struct mmc_queue *mq，下面两行从mmc_queue中获取request_queue
    struct mmc_queue *mq = d;
    struct request_queue *q = mq->queue;
    scheduler_params.sched_priority = 1;  // 设置线程优先级
    // 设置调度累为sched_fifo，实时调度，优先级为1
    sched_setscheduler(current, SCHED_FIFO, &scheduler_params); 
    // 设置紧急内存标志，说明任务很着急，一定要想办法给分配内存
    current->flags |= PF_MEMALLOC;
    down(&mq->thread_sem);
    do {
        struct request *req = NULL;
        spin_lock_irq(q->queue_lock);
        // 默认该线程休眠等待唤醒
        set_current_state(TASK_INTERRUPTIBLE);
        // 此处被唤醒，等待从gendisk的request_queue中fetch  request
        // 参数q是mmc_queue->queue,也就是struct request_queue
        // 返回的是struct request，也就是gendisk的request_queue中的request
        req = blk_fetch_request(q);
        /*
            在这里打印请求中的数据以及设备文件名，判断出错是在驱动之前还是驱动之后
            设备文件名：mmc_queue->data->disk->name
            数据：
            	request->__data_len：所有数据的长度
            	request->bio->bi_io_vec：指向bio_vec数组的指针		
            	request->bio->bio_iter->bi_sector：bio中内存数据对应的磁盘起始扇区，
            		乘以512则是磁盘的地址
            	request->bio->bi_vcnt：bio_vec数组的长度
            	page->virtual：页所对应的虚拟地址
        */
        spin_unlock_irq(q->queue_lock);
        // req代表从gendisk->request_queue中获取的request
        // mq->mqrq_prev->req代表已经获取到的request
        if (req || mq->mqrq_prev->req) {  // 有request处理
            // 有请求将自己设置为RUNNING状态
            set_current_state(TASK_RUNNING);
            // 将从gendisk中fetch的request交给mmc_queue的处理函数mmc_blk_issue_rq
            mq->issue_fn(mq, req);
        } else { // 没有请求处理，主动出让调度器
            up(&mq->thread_sem);
            schedule();
            down(&mq->thread_sem);
        }
    } while (1);
    up(&mq->thread_sem);
}
```

6、mmc_blk_issue_rq是真正的emmc驱动开始处理请求的函数  
```
int mmc_blk_issue_rq(struct mmc_queue *mq, struct request *req)
{
    struct mmc_blk_data *md = mq->data; // void *data
    struct mmc_card *card = md->queue.card;
    struct mmc_host *host = card->host;
    unsigned long flags;
    // 获取当前处理消息的命令类型
    unsigned int cmd_flags = req ? req->cmd_flags : 0;
    
    if (req && !mq->mqrq_prev->req) {
        /* claim host only for the first request */
        // 如果是第一个请求，需要获取host
        mmc_get_card(card);

        if (mmc_card_doing_bkops(host->card)) {
            ret = mmc_stop_bkops(host->card);
            if (ret)
                goto out; 
        }    
    }
    
    // 看是否有请求需要处理，如果没有直接跳转到out
    ret = mmc_blk_part_switch(card, md);
    if (ret) {
        err = mmc_blk_reset(md, card->host, MMC_BLK_PARTSWITCH);
        if (!err) {
            pr_err("%s: mmc_blk_reset(MMC_BLK_PARTSWITCH) succeeded.\n",
                    mmc_hostname(host));
            mmc_blk_reset_success(md, MMC_BLK_PARTSWITCH);
        } else 
            pr_err("%s: mmc_blk_reset(MMC_BLK_PARTSWITCH) failed.\n",
                mmc_hostname(host));

        if (req) {
            blk_end_request_all(req, -EIO);
        }
        ret = 0; 
        goto out; 
    }
    
    mmc_blk_write_packing_control(mq, req);
    
    // 根据命令类型进行对应的处理
    if (cmd_flags & REQ_DISCARD) {
    	// 擦除命令，擦除前需要先完成之前的异步请求
		/* complete ongoing async transfer before issuing discard */
        if (card->host->areq)
            mmc_blk_issue_rw_rq(mq, NULL);
        
        // 下面不管是安全擦除还是非安全擦除都要调用core提供的mmc_erase
        if (cmd_flags & REQ_SECURE &&
            !(card->quirks & MMC_QUIRK_SEC_ERASE_TRIM_BROKEN))
            ret = mmc_blk_issue_secdiscard_rq(mq, req);
        else
            ret = mmc_blk_issue_discard_rq(mq, req);
    } else if (cmd_flags & (REQ_FLUSH | REQ_BARRIER)) {
    	/* complete ongoing async transfer before issuing flush */
    	// 完成刷新缓存到磁盘的操作前也需要把之前的异步请求执行完
    	// mmc_blk_issue_rw_rq函数的第二个参数为NULL，说明没有新的请求进来，只处理就的异步请求
        if (card->host->areq)
            mmc_blk_issue_rw_rq(mq, NULL);
        /*
        	下面完成刷新缓存到磁盘的动作，会调用core层的mmc_flush_cache
        */
        ret = mmc_blk_issue_flush(mq, req);
    } else {
        /*
        	功能：EMMC处理request
        	参数：
                mq:struct mmc_queue
                req:struct request
        */
        ret = mmc_blk_issue_rw_rq(mq, req);
    }
    
out:
    if ((!req && !(test_bit(MMC_QUEUE_NEW_REQUEST, &mq->flags))) ||
         (cmd_flags & MMC_REQ_SPECIAL_MASK))
        /*
         * Release host when there are no more requests
         * and after special request(discard, flush) is done.
         * In case sepecial request, there is no reentry to
         * the 'mmc_blk_issue_rq' with 'mqrq_prev->req'.
         */
        // 释放card对应的host
        mmc_put_card(card);
    return ret;
}
```

7、上一步判断request的命令传递不同的参数给mmc_blk_issue_rw_rq完成不同的操作

```
int mmc_blk_issue_rw_rq(struct mmc_queue *mq, struct request *rqc)
{
    struct mmc_blk_data *md = mq->data;
    struct mmc_card *card = md->queue.card;
    // 代表一个mmc子系统分装的request
    struct mmc_blk_request *brq = &mq->mqrq_cur->brq;
    int ret = 1, disable_multi = 0, retry = 0, type, retune_retry_done = 0;
    enum mmc_blk_status status;
    struct mmc_queue_req *mq_rq;
    struct request *req = rqc;
    struct mmc_async_req *areq;
    const u8 packed_nr = 2;
    u8 reqs = 0;
    bool reset = false;
#ifdef CONFIG_MMC_SIMULATE_MAX_SPEED
    unsigned long waitfor = jiffies;
#endif
	
	// 如果没有请求，队列也是空的，直接返回0
    if (!rqc && !mq->mqrq_prev->req)
        return 0;
    
    // 如果有新的请求，把新的请求加入mmc_queue的request_queue中
    if (rqc) {
    	reqs = mmc_blk_prep_packed_list(mq, rqc);
    }
    
    // 下面循环从请求队列中获取请求调用mmc_start_req处理请求，该函数是core提供的
    do {
    	if (rqc) {
            /*
             * When 4KB native sector is enabled, only 8 blocks
             * multiple read or write is allowed
             */
            if ((brq->data.blocks & 0x07) &&
                (card->ext_csd.data_sector_size == 4096)) {
                pr_err("%s: Transfer size is not 4KB sector size aligned\n",
                    req->rq_disk->disk_name);
                mq_rq = mq->mqrq_cur;
                goto cmd_abort;
            }

            if (reqs >= packed_nr)
                mmc_blk_packed_hdr_wrq_prep(mq->mqrq_cur,
                                card, mq);
            else
                mmc_blk_rw_rq_prep(mq->mqrq_cur, card, 0, mq);
            areq = &mq->mqrq_cur->mmc_active;
        } else
            areq = NULL;
        
        // 调用core层接口发起非阻塞请求处理 
        areq = mmc_start_req(card->host, areq, (int *) &status);
        if (!areq) {
            if (status == MMC_BLK_NEW_REQUEST)
                set_bit(MMC_QUEUE_NEW_REQUEST, &mq->flags);
            return 0;
        }
		
        mq_rq = container_of(areq, struct mmc_queue_req, mmc_active);
        brq = &mq_rq->brq;
        req = mq_rq->req;
        type = rq_data_dir(req) == READ ? MMC_BLK_READ : MMC_BLK_WRITE;
        mmc_queue_bounce_post(mq_rq);
			
        if (card->err_in_sdr104) {
            /*
             * Data CRC/timeout errors will manifest as CMD/DATA
             * ERR. But we'd like to retry these too.
             * Moreover, no harm done if this fails too for multiple
             * times, we anyway reduce the bus-speed and retry the
             * same request.
             * If that fails too, we don't override this status.
             */
            if (status == MMC_BLK_ABORT ||
                status == MMC_BLK_CMD_ERR ||
                status == MMC_BLK_DATA_ERR ||
                status == MMC_BLK_RETRY)
                /* reset on all of these errors and retry */
                reset = true;

            status = MMC_BLK_RETRY;
            card->err_in_sdr104 = false;
        }

		// 下面都是返回状态判断，异常处理
        switch (status) {
        case MMC_BLK_SUCCESS:
        case MMC_BLK_PARTIAL:
            /*
             * A block was successfully transferred.
             */
            mmc_blk_reset_success(md, type);

            mmc_blk_simulate_delay(mq, rqc, waitfor);

            if (mmc_packed_cmd(mq_rq->cmd_type)) {
                ret = mmc_blk_end_packed_req(mq_rq);
                break;
            } else {
                ret = blk_end_request(req, 0,brq->data.bytes_xfered);
            }

            /*
             * If the blk_end_request function returns non-zero even
             * though all data has been transferred and no errors
             * were returned by the host controller, it's a bug.
             */
            if (status == MMC_BLK_SUCCESS && ret) {
                pr_err("%s BUG rq_tot %d d_xfer %d\n",
                       __func__, blk_rq_bytes(req),
                       brq->data.bytes_xfered);
                rqc = NULL;
                goto cmd_abort;
            }
            break;
        case MMC_BLK_CMD_ERR:
            ret = mmc_blk_cmd_err(md, card, brq, req, ret);
            if (mmc_blk_reset(md, card->host, type))
                goto cmd_abort;
            if (!ret)
                goto start_new_req;
            break;
        case MMC_BLK_RETRY:
            retune_retry_done = brq->retune_retry_done;
            if (retry++ < MMC_BLK_MAX_RETRIES) {
                break;
            } else if (reset) {
                reset = false;
                /*
                 * If we exhaust all the retries due to
                 * CRC/timeout errors in SDR140 mode with UHS SD
                 * cards, re-configure the card in SDR50
                 * bus-speed mode.
                 * All subsequent re-init of this card will be
                 * in SDR50 mode, unless it is removed and
                 * re-inserted. When new UHS SD cards are
                 * inserted, it may start at SDR104 mode if
                 * supported by the card.
                 */
                pr_err("%s: blocked SDR104, lower the bus-speed (SDR50 / DDR50)\n",
                    req->rq_disk->disk_name);
                mmc_host_clear_sdr104(card->host);
                mmc_suspend_clk_scaling(card->host);
                mmc_blk_reset(md, card->host, type);
                /* SDR104 mode is blocked from now on */
                card->sdr104_blocked = true;
                /* retry 5 times again */
                retry = 0;
                break;
            }
            /* Fall through */
        case MMC_BLK_ABORT:
            if (!mmc_blk_reset(md, card->host, type) &&
                (retry++ < (MMC_BLK_MAX_RETRIES + 1)))
                break;
            goto cmd_abort;
        case MMC_BLK_DATA_ERR: {
            int err;

            err = mmc_blk_reset(md, card->host, type);
            if (!err)
                break;
            goto cmd_abort;
        }
        case MMC_BLK_ECC_ERR:
            if (brq->data.blocks > 1) {
                /* Redo read one sector at a time */
                pr_warn("%s: retrying using single block read\n",
                    req->rq_disk->disk_name);
                disable_multi = 1;
                break;
            }
            /*
             * After an error, we redo I/O one sector at a
             * time, so we only reach here after trying to
             * read a single sector.
             */
            ret = blk_end_request(req, -EIO,
                        brq->data.blksz);
            if (!ret)
                goto start_new_req;
            break;
        case MMC_BLK_NOMEDIUM:
            goto cmd_abort;
        default:
            pr_err("%s: Unhandled return value (%d)",
                    req->rq_disk->disk_name, status);
            goto cmd_abort;
        }

        if (ret) {
            if (mmc_packed_cmd(mq_rq->cmd_type)) {
                if (!mq_rq->packed->retries)
                    goto cmd_abort;
                mmc_blk_packed_hdr_wrq_prep(mq_rq, card, mq);
                mmc_start_req(card->host,&mq_rq->mmc_active, NULL);
             } else {
                    /*
                     * In case of a incomplete request
                     * prepare it again and resend.
                     */
                    mmc_blk_rw_rq_prep(mq_rq, card, disable_multi, mq);
                    mmc_start_req(card->host, &mq_rq->mmc_active, NULL);
              }
              mq_rq->brq.retune_retry_done = retune_retry_done;
        }
    } while (ret);

    return 1;
    
cmd_abort:
    if (mmc_packed_cmd(mq_rq->cmd_type)) {
        mmc_blk_abort_packed_req(mq_rq);
    } else {
        if (mmc_card_removed(card))
            req->cmd_flags |= REQ_QUIET;
        while (ret)
            ret = blk_end_request(req, -EIO,
                    blk_rq_cur_bytes(req));
    }

start_new_req:
    if (rqc) {
        if (mmc_card_removed(card)) {
            rqc->cmd_flags |= REQ_QUIET;
            blk_end_request_all(rqc, -EIO);
        } else {
            /*
             * If current request is packed, it needs to put back.
             */
            if (mmc_packed_cmd(mq->mqrq_cur->cmd_type))
                mmc_blk_revert_packed_req(mq, mq->mqrq_cur);

            mmc_blk_rw_rq_prep(mq->mqrq_cur, card, 0, mq);
            mmc_start_req(card->host,
                      &mq->mqrq_cur->mmc_active, NULL);
        }
    }

    return 0;
}
```

8、MMC驱动开始使用非阻塞方式处理request，

```
/*
	定义：driver/mmc/core/core.c
	返回值：已经完成的非阻塞请求
*/
struct mmc_async_req *mmc_start_req(struct mmc_host *host,
					struct mmc_async_req *areq, int *error)
{
	// 如果之前没有发起的非阻塞请求，则创建一个非阻塞请求
	if (areq) {
		mmc_pre_req(host, areq->mrq, !host->areq);
	}
	
	if (!err && areq) { 
		trace_mmc_blk_rw_start(areq->mrq->cmd->opcode,
								areq->mrq->cmd->arg,areq->mrq->data);
		// 交给mmc core处理请求
        __mmc_start_data_req(host, areq->mrq);
	}
}
```

9、进入mmc/core/core.c，处理请求数据

```
int __mmc_start_data_req(struct mmc_host *host, struct mmc_request *mrq)
{
	int err;
    mrq->done = mmc_wait_data_done;	// 完成请求的回调函数
    mrq->host = host;
    err = mmc_start_request(host, mrq); // 还是没有真正发命令，快了
    if (err) {
    	// 如果出错，直接等待处理结束，否则返回
        mrq->cmd->error = err;
        mmc_wait_data_done(mrq);
    }
    return err;
}

int mmc_start_request(struct mmc_host *host, struct mmc_request *mrq)
{
	// 上面各种异常判断，下面调用真正的数据处理
	__mmc_start_request(host, mrq);
}

void __mmc_start_request(struct mmc_host *host, struct mmc_request *mrq)
{
	// 省略异常判断，下面终于到了最终要的时候
	/*
		1、此处就要介绍到高通的host controller了，controller是直接通过mmc物理总			
		线连接EMMC内部的MMC接口（cmd、data0~7、clk）
		2、mmc子系统使用core层管理card和host，在host层又抽象出sdhc标准，高通的			
		msm系列和三星的s3c系列都使用了该标准，高通msm系列的实现在sdhci-msm.c中
		3、sdhci抽象出host公共部分，vendor只需要实现pinctl、power等自己特有的部		
		分，然后实现sdhci提供的接口
		4、下面的host代表高通的符合sdhci标准的host，core层交给host处理读写数据的			
		api是sdhci_request，也就是host->ops->request，在host驱动初始化时会设置
		，就是在sdhci-msm.c中设置的
	*/
	host->ops->request(host, mrq);
}
```

10、千呼万唤始出来，终于到了host controller了，开始真正的和物理MMC卡交互了  
mmc/host/sdhci.c

```
void sdhci_request(struct mmc_host *mmc, struct mmc_request *mrq)
{
	struct sdhci_host *host;	// sdhci标准抽象的host
	int present;
    unsigned long flags;
    /*
    	host关系：mmc_host->sdhci_host
    */
	host = mmc_priv(mmc);	// 从mmc_host的private成员就是sdhci_host
	
	spin_lock_irqsave(&host->lock, flags);
	// ...中间一些异常判断
	
	// struct mmc_request *mrq;
	host->mrq = mrq;
	
	if (!present || host->flags & SDHCI_DEVICE_DEAD) {
		// 异常处理，此处只分析正常读写
	} else {
		// 略过加密传输
		if (mrq->sbc && !(host->flags & SDHCI_AUTO_CMD23)) {
			sdhci_send_command(host, mrq->sbc);
		} else {
			// 正常写数据应该走这边，这个函数才是真正的将request中的数据转换成mm命令
			sdhci_send_command(host, mrq->cmd);
		}
	}
	mmiowb();	// io屏障，防止乱序执行
    spin_unlock_irqrestore(&host->lock, flags);
    return;
}
```

11、不，这才是真正的发送MMC命令  

```
void sdhci_send_command(struct sdhci_host *host, struct mmc_command *cmd)
{
	int flags;
    u32 mask;
    unsigned long timeout;
    timeout = 10000;	// 10ms
    mask = SDHCI_CMD_INHIBIT;  // 默认设置传输命令
    if ((cmd->data != NULL) || (cmd->flags & MMC_RSP_BUSY)) {
    	// 数据传输走这里
    	mask |= SDHCI_DATA_INHIBIT;
    }
    
    // ...中间判断了是否超时
    
    host->cmd = cmd;
    host->busy_handle = 0;
    // 准备DMA，使用了scatterlist内存分散列表
    sdhci_prepare_data(host, cmd);
    sdhci_writel(host, cmd->arg, SDHCI_ARGUMENT);
    sdhci_set_transfer_mode(host, cmd);
    trace_mmc_cmd_rw_start(cmd->opcode, cmd->arg, cmd->flags);
    // 通过该命令触发DMA传输数据
    sdhci_writew(host, SDHCI_MAKE_CMD(cmd->opcode, flags), SDHCI_COMMAND);
}
```

12、到这里不得不疑问，要处理的数据，page对应的data在哪里
```
void sdhci_prepare_data(struct sdhci_host *host, struct mmc_command *cmd)
{
	// mmc_data承载了要传输的数据
	struct mmc_data *data = cmd->data;
	
	if (host->flags & SDHCI_REQ_USE_DMA) { // 使用DMA
		if (host->flags & SDHCI_USE_ADMA) {	// 使用ADMA，高级DMA
			trace_mmc_adma_table_pre(cmd->opcode, data->sg_len);
			ret = sdhci_adma_table_pre(host, data);
		} else { // 可能走这里
			int sg_cnt;
			sg_cnt = sdhci_pre_dma_transfer(host, data);
			// 此处忽略sg_cnt失败情况
			sdhci_writel(host, sg_dma_address(data->sg),
                    SDHCI_DMA_ADDRESS);
		}
	}
	
	sdhci_set_transfer_irqs(host);
	sdhci_set_blk_size_reg(host, data->blksz, SDHCI_DEFAULT_BOUNDARY_ARG);
	sdhci_writew(host, data->blocks, SDHCI_BLOCK_COUNT);
}

int sdhci_pre_dma_transfer(struct sdhci_host *host,struct mmc_data *data)
{
	/*
		mmc_data->sg包含了data在内存中映射
		想要打印传输给EMMC的数据该怎么办：
			mmc_data->sg->dma_address是内存DMA起始地址
			mmc_data->sg->dma_length是DMA内存的长度
			上面一组参数代表了一个sg，但是sg是一个list
			mmc_data->sg_count记录了sg的数量
			所以整体要写入的数据应该是遍历整个mmc_data包含的sg数组
				mmc_data->sg[0]～mmc_data->sg[sg_count-1]
		但是什么时候打印呢，如何知道打印的是写哪个分区的
			1、什么时候打印
				在设置了dma方式的时候，mmc_data中sg的数组中每一个sg代表一段数据地址空间
				只要遍历sg数组即可
			2、怎么判断打印的是哪个分区（UDA的分区mmcblk0px）
				mmc_data->mmc_request->mmc_host->mmc_card->mmc_part
	*/
	int sg_count;
	sg_count = dma_map_sg(mmc_dev(host->mmc), data->sg, data->sg_len,
                data->flags & MMC_DATA_WRITE ?
                DMA_TO_DEVICE : DMA_FROM_DEVICE);
    return sg_count;
}     
```
###　2、HOST

这里介绍高通的sdhci-msm.c中host的实现，该host是符合sdhci标准的

https://blog.csdn.net/ooonebook/article/details/55272522

1、注册host driver ( drivers/mmc/host )

```
static struct platform_driver sdhci_msm_driver = {
    .probe      = sdhci_msm_probe,
    .remove     = sdhci_msm_remove,
    .driver     = {  
        .name   = "sdhci_msm",
        .owner  = THIS_MODULE,
        .probe_type = PROBE_PREFER_ASYNCHRONOUS,
        .of_match_table = sdhci_msm_dt_match,
        .pm = SDHCI_MSM_PMOPS,
    },   
};
module_platform_driver(sdhci_msm_driver);
```

2、当host驱动注册的时候mmc bus会无条件执行sdhci_msm_driver->sdhci_msm_probe

```
static int sdhci_msm_probe(struct platform_device *pdev)
{   
    const struct sdhci_msm_offset *msm_host_offset;
    struct sdhci_host *host;
    struct sdhci_pltfm_host *pltfm_host;
    struct sdhci_msm_host *msm_host;
    struct resource *core_memres = NULL;
    int ret = 0, dead = 0;
    u16 host_version;
    u32 irq_status, irq_ctl;
    struct resource *tlmm_memres = NULL;
    void __iomem *tlmm_mem;
    unsigned long flags;
    bool force_probe;
    char boot_marker[40];
    
    pr_debug("%s: Enter %s\n", dev_name(&pdev->dev), __func__);
    msm_host = devm_kzalloc(&pdev->dev, sizeof(struct sdhci_msm_host),
                GFP_KERNEL);
    if (!msm_host) {
        ret = -ENOMEM;
        goto out;
    }

    if (of_find_compatible_node(NULL, NULL, "qcom,sdhci-msm-v5")) {
        msm_host->mci_removed = true;
        msm_host->offset = &sdhci_msm_offset_mci_removed;
    } else {
        msm_host->mci_removed = false;
        msm_host->offset = &sdhci_msm_offset_mci_present;
    }
    msm_host_offset = msm_host->offset;
    msm_host->sdhci_msm_pdata.ops = &sdhci_msm_ops;
    
    // 下面会分配mmc_host，mmc_host中包含了const struct mmc_host_ops *ops;
    // 上面的mmc_host_ops中包含了core层调用host处理request的回调函数
    // 在sdhci.c中定义了struct mmc_host_ops sdhci_ops，request成员初始化为
    // sdhci_request
    host = sdhci_pltfm_init(pdev, &msm_host->sdhci_msm_pdata, 0);
    if (IS_ERR(host)) {
    	ret = PTR_ERR(host);
        goto out_host_free;
    }

    snprintf(boot_marker, sizeof(boot_marker),
            "M - DRIVER %s Init", mmc_hostname(host->mmc));
    place_marker(boot_marker);

    pltfm_host = sdhci_priv(host);
    pltfm_host->priv = msm_host;
    msm_host->mmc = host->mmc;
    msm_host->pdev = pdev;

    /* get the ice device vops if present */
    ret = sdhci_msm_ice_get_dev(host);
    if (ret == -EPROBE_DEFER) {
        /*
         * SDHCI driver might be probed before ICE driver does.
         * In that case we would like to return EPROBE_DEFER code
         * in order to delay its probing.
         */
        dev_err(&pdev->dev, "%s: required ICE device not probed yet err = %d\n",
            __func__, ret);
        goto pltfm_free;

    } else if (ret == -ENODEV) {
        /*
         * ICE device is not enabled in DTS file. No need for further
         * initialization of ICE driver.
         */
         dev_warn(&pdev->dev, "%s: ICE device is not enabled",
            __func__);
    } else if (ret) {
        dev_err(&pdev->dev, "%s: sdhci_msm_ice_get_dev failed %d\n",
            __func__, ret);
        goto pltfm_free;
    }

    /* Extract platform data */
    if (pdev->dev.of_node) {
        ret = of_alias_get_id(pdev->dev.of_node, "sdhc");
        if (ret <= 0) {
            dev_err(&pdev->dev, "Failed to get slot index %d\n",
                ret);
            goto pltfm_free;
        }

        /* Read property to determine if the probe is forced */
        force_probe = of_find_property(pdev->dev.of_node,
            "qcom,force-sdhc1-probe", NULL);

        /* skip the probe if eMMC isn't a boot device */
        if ((ret == 1) && !sdhci_msm_is_bootdevice(&pdev->dev)
            && !force_probe) {
            ret = -ENODEV;
            goto pltfm_free;
        }
        
        if (disable_slots & (1 << (ret - 1))) {
            dev_info(&pdev->dev, "%s: Slot %d disabled\n", __func__,
                ret);
            ret = -ENODEV;
            goto pltfm_free;
        }

        if (ret <= 2)
            sdhci_slot[ret-1] = msm_host;

        msm_host->pdata = sdhci_msm_populate_pdata(&pdev->dev,
                               msm_host);
        if (!msm_host->pdata) {
            dev_err(&pdev->dev, "DT parsing error\n");
            goto pltfm_free;
        }
    } else {
        dev_err(&pdev->dev, "No device tree node\n");
        goto pltfm_free;
    }
    
    /* Setup Clocks */

    /* Setup SDCC bus voter clock. */
    msm_host->bus_clk = devm_clk_get(&pdev->dev, "bus_clk");
    if (!IS_ERR_OR_NULL(msm_host->bus_clk)) {
        /* Vote for max. clk rate for max. performance */
        ret = clk_set_rate(msm_host->bus_clk, INT_MAX);
        if (ret)
            goto pltfm_free;
        ret = clk_prepare_enable(msm_host->bus_clk);
        if (ret)
            goto pltfm_free;
    }

    /* Setup main peripheral bus clock */
    msm_host->pclk = devm_clk_get(&pdev->dev, "iface_clk");
    if (!IS_ERR(msm_host->pclk)) {
        ret = clk_prepare_enable(msm_host->pclk);
        if (ret)
            goto bus_clk_disable;
    }
    atomic_set(&msm_host->controller_clock, 1);

    if (msm_host->ice.pdev) {
        /* Setup SDC ICE clock */
        msm_host->ice_clk = devm_clk_get(&pdev->dev, "ice_core_clk");
        if (!IS_ERR(msm_host->ice_clk)) {
            /* ICE core has only one clock frequency for now */
            ret = clk_set_rate(msm_host->ice_clk,
                    msm_host->pdata->ice_clk_max);
            if (ret) {
                dev_err(&pdev->dev, "ICE_CLK rate set failed (%d) for %u\n",
                    ret,
                    msm_host->pdata->ice_clk_max);
                goto pclk_disable;
            }
            ret = clk_prepare_enable(msm_host->ice_clk);
            if (ret)
                goto pclk_disable;

            msm_host->ice_clk_rate = msm_host->pdata->ice_clk_max;
        }
    }
	/* Setup SDC MMC clock */
    msm_host->clk = devm_clk_get(&pdev->dev, "core_clk");
    if (IS_ERR(msm_host->clk)) {
        ret = PTR_ERR(msm_host->clk);
        goto pclk_disable;
    }

    /* Set to the minimum supported clock frequency */
    ret = clk_set_rate(msm_host->clk, sdhci_msm_get_min_clock(host));
    if (ret) {
        dev_err(&pdev->dev, "MClk rate set failed (%d)\n", ret);
        goto pclk_disable;
    }
    ret = clk_prepare_enable(msm_host->clk);
    if (ret)
        goto pclk_disable;

    msm_host->clk_rate = sdhci_msm_get_min_clock(host);
    atomic_set(&msm_host->clks_on, 1);

    /* Setup CDC calibration fixed feedback clock */
    msm_host->ff_clk = devm_clk_get(&pdev->dev, "cal_clk");
    if (!IS_ERR(msm_host->ff_clk)) {
        ret = clk_prepare_enable(msm_host->ff_clk);
        if (ret)
            goto clk_disable;
    }

    /* Setup CDC calibration sleep clock */
    msm_host->sleep_clk = devm_clk_get(&pdev->dev, "sleep_clk");
    if (!IS_ERR(msm_host->sleep_clk)) {
        ret = clk_prepare_enable(msm_host->sleep_clk);
        if (ret)
            goto ff_clk_disable;
    }
    
    msm_host->saved_tuning_phase = INVALID_TUNING_PHASE;

    ret = sdhci_msm_bus_register(msm_host, pdev);
    if (ret)
        goto sleep_clk_disable;

    if (msm_host->msm_bus_vote.client_handle)
        INIT_DELAYED_WORK(&msm_host->msm_bus_vote.vote_work,
                  sdhci_msm_bus_work);
    sdhci_msm_bus_voting(host, 1);

    /* Setup regulators */
    ret = sdhci_msm_vreg_init(&pdev->dev, msm_host->pdata, true);
    if (ret) {
        dev_err(&pdev->dev, "Regulator setup failed (%d)\n", ret);
        goto bus_unregister;
    }

    /* Reset the core and Enable SDHC mode */
    core_memres = platform_get_resource_byname(pdev,
                IORESOURCE_MEM, "core_mem");
    if (!msm_host->mci_removed) {
        if (!core_memres) {
            dev_err(&pdev->dev, "Failed to get iomem resource\n");
            goto vreg_deinit;
        }
        msm_host->core_mem = devm_ioremap(&pdev->dev,
            core_memres->start, resource_size(core_memres));

        if (!msm_host->core_mem) {
            dev_err(&pdev->dev, "Failed to remap registers\n");
            ret = -ENOMEM;
            goto vreg_deinit;
        }
    }
    
    tlmm_memres = platform_get_resource_byname(pdev,
                IORESOURCE_MEM, "tlmm_mem");
    if (tlmm_memres) {
        tlmm_mem = devm_ioremap(&pdev->dev, tlmm_memres->start,
                        resource_size(tlmm_memres));

        if (!tlmm_mem) {
            dev_err(&pdev->dev, "Failed to remap tlmm registers\n");
            ret = -ENOMEM;
            goto vreg_deinit;
        }
        writel_relaxed(readl_relaxed(tlmm_mem) | 0x2, tlmm_mem);
    }

    /*
     * Reset the vendor spec register to power on reset state.
     */
    writel_relaxed(CORE_VENDOR_SPEC_POR_VAL,
    host->ioaddr + msm_host_offset->CORE_VENDOR_SPEC);

    /*
     * Ensure SDHCI FIFO is enabled by disabling alternative FIFO
     */
    writel_relaxed((readl_relaxed(host->ioaddr +
            msm_host_offset->CORE_VENDOR_SPEC3) &
            ~CORE_FIFO_ALT_EN), host->ioaddr +
            msm_host_offset->CORE_VENDOR_SPEC3);

    if (!msm_host->mci_removed) {
        /* Set HC_MODE_EN bit in HC_MODE register */
        writel_relaxed(HC_MODE_EN, (msm_host->core_mem + CORE_HC_MODE));

        /* Set FF_CLK_SW_RST_DIS bit in HC_MODE register */
        writel_relaxed(readl_relaxed(msm_host->core_mem +
                CORE_HC_MODE) | FF_CLK_SW_RST_DIS,
                msm_host->core_mem + CORE_HC_MODE);
    }
    sdhci_set_default_hw_caps(msm_host, host);
    
    /*
     * Set the PAD_PWR_SWTICH_EN bit so that the PAD_PWR_SWITCH bit can
     * be used as required later on.
     */
    writel_relaxed((readl_relaxed(host->ioaddr +
            msm_host_offset->CORE_VENDOR_SPEC) |
            CORE_IO_PAD_PWR_SWITCH_EN), host->ioaddr +
            msm_host_offset->CORE_VENDOR_SPEC);
    /*
     * CORE_SW_RST above may trigger power irq if previous status of PWRCTL
     * was either BUS_ON or IO_HIGH_V. So before we enable the power irq
     * interrupt in GIC (by registering the interrupt handler), we need to
     * ensure that any pending power irq interrupt status is acknowledged
     * otherwise power irq interrupt handler would be fired prematurely.
     */
    irq_status = sdhci_msm_readl_relaxed(host,
        msm_host_offset->CORE_PWRCTL_STATUS);
    sdhci_msm_writel_relaxed(irq_status, host,
        msm_host_offset->CORE_PWRCTL_CLEAR);
    irq_ctl = sdhci_msm_readl_relaxed(host,
        msm_host_offset->CORE_PWRCTL_CTL);

    if (irq_status & (CORE_PWRCTL_BUS_ON | CORE_PWRCTL_BUS_OFF))
        irq_ctl |= CORE_PWRCTL_BUS_SUCCESS;
    if (irq_status & (CORE_PWRCTL_IO_HIGH | CORE_PWRCTL_IO_LOW))
        irq_ctl |= CORE_PWRCTL_IO_SUCCESS;
    sdhci_msm_writel_relaxed(irq_ctl, host,
        msm_host_offset->CORE_PWRCTL_CTL);

    /*
     * Ensure that above writes are propogated before interrupt enablement
     * in GIC.
     */
    mb();
    
    /*
     * Following are the deviations from SDHC spec v3.0 -
     * 1. Card detection is handled using separate GPIO.
     * 2. Bus power control is handled by interacting with PMIC.
     */
    host->quirks |= SDHCI_QUIRK_BROKEN_CARD_DETECTION;
    host->quirks |= SDHCI_QUIRK_SINGLE_POWER_WRITE;
    host->quirks |= SDHCI_QUIRK_CAP_CLOCK_BASE_BROKEN;
    host->quirks |= SDHCI_QUIRK_NO_ENDATTR_IN_NOPDESC;
    host->quirks2 |= SDHCI_QUIRK2_ALWAYS_USE_BASE_CLOCK;
    host->quirks2 |= SDHCI_QUIRK2_IGNORE_DATATOUT_FOR_R1BCMD;
    host->quirks2 |= SDHCI_QUIRK2_BROKEN_PRESET_VALUE;
    host->quirks2 |= SDHCI_QUIRK2_USE_RESERVED_MAX_TIMEOUT;
    host->quirks2 |= SDHCI_QUIRK2_NON_STANDARD_TUNING;
    host->quirks2 |= SDHCI_QUIRK2_USE_PIO_FOR_EMMC_TUNING;

    if (host->quirks2 & SDHCI_QUIRK2_ALWAYS_USE_BASE_CLOCK)
        host->quirks2 |= SDHCI_QUIRK2_DIVIDE_TOUT_BY_4;

    host_version = readw_relaxed((host->ioaddr + SDHCI_HOST_VERSION));
    dev_dbg(&pdev->dev, "Host Version: 0x%x Vendor Version 0x%x\n",
        host_version, ((host_version & SDHCI_VENDOR_VER_MASK) >>
          SDHCI_VENDOR_VER_SHIFT));
    if (((host_version & SDHCI_VENDOR_VER_MASK) >>
        SDHCI_VENDOR_VER_SHIFT) == SDHCI_VER_100) {
        /*
         * Add 40us delay in interrupt handler when
         * operating at initialization frequency(400KHz).
         */
        host->quirks2 |= SDHCI_QUIRK2_SLOW_INT_CLR;
        /*
         * Set Software Reset for DAT line in Software
         * Reset Register (Bit 2).
         */
        host->quirks2 |= SDHCI_QUIRK2_RDWR_TX_ACTIVE_EOT;
    }

    host->quirks2 |= SDHCI_QUIRK2_IGN_DATA_END_BIT_ERROR;
    
    /* Setup PWRCTL irq */
    msm_host->pwr_irq = platform_get_irq_byname(pdev, "pwr_irq");
    if (msm_host->pwr_irq < 0) {
        dev_err(&pdev->dev, "Failed to get pwr_irq by name (%d)\n",
                msm_host->pwr_irq);
        goto vreg_deinit;
    }
    ret = devm_request_threaded_irq(&pdev->dev, msm_host->pwr_irq, NULL,
                    sdhci_msm_pwr_irq, IRQF_ONESHOT,
                    dev_name(&pdev->dev), host);
    if (ret) {
        dev_err(&pdev->dev, "Request threaded irq(%d) failed (%d)\n",
                msm_host->pwr_irq, ret);
        goto vreg_deinit;
    }

    /* Enable pwr irq interrupts */
    sdhci_msm_writel_relaxed(INT_MASK, host,
        msm_host_offset->CORE_PWRCTL_MASK);

#ifdef CONFIG_MMC_CLKGATE
    /* Set clock gating delay to be used when CONFIG_MMC_CLKGATE is set */
    msm_host->mmc->clkgate_delay = SDHCI_MSM_MMC_CLK_GATE_DELAY;
#endif
	
	/* Set host capabilities */
    msm_host->mmc->caps |= msm_host->pdata->mmc_bus_width;
    msm_host->mmc->caps |= msm_host->pdata->caps;
    msm_host->mmc->caps |= MMC_CAP_AGGRESSIVE_PM;
    msm_host->mmc->caps |= MMC_CAP_WAIT_WHILE_BUSY;
    msm_host->mmc->caps2 |= msm_host->pdata->caps2;
    msm_host->mmc->caps2 |= MMC_CAP2_BOOTPART_NOACC;
    msm_host->mmc->caps2 |= MMC_CAP2_HS400_POST_TUNING;
    msm_host->mmc->caps2 |= MMC_CAP2_CLK_SCALE;
    msm_host->mmc->caps2 |= MMC_CAP2_SANITIZE;
    msm_host->mmc->caps2 |= MMC_CAP2_MAX_DISCARD_SIZE;
    msm_host->mmc->caps2 |= MMC_CAP2_SLEEP_AWAKE;
    msm_host->mmc->pm_caps |= MMC_PM_KEEP_POWER | MMC_PM_WAKE_SDIO_IRQ;

    if (msm_host->pdata->nonremovable)
        msm_host->mmc->caps |= MMC_CAP_NONREMOVABLE;

    if (msm_host->pdata->nonhotplug)
        msm_host->mmc->caps2 |= MMC_CAP2_NONHOTPLUG;

    msm_host->mmc->sdr104_wa = msm_host->pdata->sdr104_wa;

    /* Initialize ICE if present */
    if (msm_host->ice.pdev) {
        ret = sdhci_msm_ice_init(host);
        if (ret) {
            dev_err(&pdev->dev, "%s: SDHCi ICE init failed (%d)\n",
                    mmc_hostname(host->mmc), ret);
            ret = -EINVAL;
            goto vreg_deinit;
        }
        host->is_crypto_en = true;
        /* Packed commands cannot be encrypted/decrypted using ICE */
        msm_host->mmc->caps2 &= ~(MMC_CAP2_PACKED_WR |
                MMC_CAP2_PACKED_WR_CONTROL);
    }
    
    init_completion(&msm_host->pwr_irq_completion);

    if (gpio_is_valid(msm_host->pdata->status_gpio)) {
        /*
         * Set up the card detect GPIO in active configuration before
         * configuring it as an IRQ. Otherwise, it can be in some
         * weird/inconsistent state resulting in flood of interrupts.
         */
        sdhci_msm_setup_pins(msm_host->pdata, true);

        /*
         * This delay is needed for stabilizing the card detect GPIO
         * line after changing the pull configs.
         */
        usleep_range(10000, 10500);
        ret = mmc_gpio_request_cd(msm_host->mmc,
                msm_host->pdata->status_gpio, 0);
        if (ret) {
            dev_err(&pdev->dev, "%s: Failed to request card detection IRQ %d\n",
                    __func__, ret);
            goto vreg_deinit;
        }
    }

    if ((sdhci_readl(host, SDHCI_CAPABILITIES) & SDHCI_CAN_64BIT) &&
        (dma_supported(mmc_dev(host->mmc), DMA_BIT_MASK(64)))) {
        host->dma_mask = DMA_BIT_MASK(64);
        mmc_dev(host->mmc)->dma_mask = &host->dma_mask;
        mmc_dev(host->mmc)->coherent_dma_mask  = host->dma_mask;
    } else if (dma_supported(mmc_dev(host->mmc), DMA_BIT_MASK(32))) {
        host->dma_mask = DMA_BIT_MASK(32);
        mmc_dev(host->mmc)->dma_mask = &host->dma_mask;
        mmc_dev(host->mmc)->coherent_dma_mask  = host->dma_mask;
    } else {
        dev_err(&pdev->dev, "%s: Failed to set dma mask\n", __func__);
    }
    
     msm_host->pdata->sdiowakeup_irq = platform_get_irq_byname(pdev,
                              "sdiowakeup_irq");
    if (sdhci_is_valid_gpio_wakeup_int(msm_host)) {
        dev_info(&pdev->dev, "%s: sdiowakeup_irq = %d\n", __func__,
                msm_host->pdata->sdiowakeup_irq);
        msm_host->is_sdiowakeup_enabled = true;
        ret = request_irq(msm_host->pdata->sdiowakeup_irq,
                  sdhci_msm_sdiowakeup_irq,
                  IRQF_SHARED | IRQF_TRIGGER_HIGH,
                  "sdhci-msm sdiowakeup", host);
        if (ret) {
            dev_err(&pdev->dev, "%s: request sdiowakeup IRQ %d: failed: %d\n",
                __func__, msm_host->pdata->sdiowakeup_irq, ret);
            msm_host->pdata->sdiowakeup_irq = -1;
            msm_host->is_sdiowakeup_enabled = false;
            goto vreg_deinit;
        } else {
            spin_lock_irqsave(&host->lock, flags);
            sdhci_msm_cfg_sdiowakeup_gpio_irq(host, false);
            msm_host->sdio_pending_processing = false;
            spin_unlock_irqrestore(&host->lock, flags);
        }
    }

    sdhci_msm_cmdq_init(host, pdev);
    ret = sdhci_add_host(host);
    if (ret) {
        dev_err(&pdev->dev, "Add host failed (%d)\n", ret);
        goto vreg_deinit;
    }

    msm_host->pltfm_init_done = true;

    pm_runtime_set_active(&pdev->dev);
    pm_runtime_enable(&pdev->dev);
    pm_runtime_set_autosuspend_delay(&pdev->dev, MSM_AUTOSUSPEND_DELAY_MS);
    pm_runtime_use_autosuspend(&pdev->dev);
    
    msm_host->msm_bus_vote.max_bus_bw.show = show_sdhci_max_bus_bw;
    msm_host->msm_bus_vote.max_bus_bw.store = store_sdhci_max_bus_bw;
    sysfs_attr_init(&msm_host->msm_bus_vote.max_bus_bw.attr);
    msm_host->msm_bus_vote.max_bus_bw.attr.name = "max_bus_bw";
    msm_host->msm_bus_vote.max_bus_bw.attr.mode = S_IRUGO | S_IWUSR;
    ret = device_create_file(&pdev->dev,
            &msm_host->msm_bus_vote.max_bus_bw);
    if (ret)
        goto remove_host;

    if (!gpio_is_valid(msm_host->pdata->status_gpio)) {
        msm_host->polling.show = show_polling;
        msm_host->polling.store = store_polling;
        sysfs_attr_init(&msm_host->polling.attr);
        msm_host->polling.attr.name = "polling";
        msm_host->polling.attr.mode = S_IRUGO | S_IWUSR;
        ret = device_create_file(&pdev->dev, &msm_host->polling);
        if (ret)
            goto remove_max_bus_bw_file;
    }

    msm_host->auto_cmd21_attr.show = show_auto_cmd21;
    msm_host->auto_cmd21_attr.store = store_auto_cmd21;
    sysfs_attr_init(&msm_host->auto_cmd21_attr.attr);
    msm_host->auto_cmd21_attr.attr.name = "enable_auto_cmd21";
    msm_host->auto_cmd21_attr.attr.mode = S_IRUGO | S_IWUSR;
    ret = device_create_file(&pdev->dev, &msm_host->auto_cmd21_attr);
    if (ret) {
        pr_err("%s: %s: failed creating auto-cmd21 attr: %d\n",
               mmc_hostname(host->mmc), __func__, ret);
        device_remove_file(&pdev->dev, &msm_host->auto_cmd21_attr);
    }
    if (sdhci_msm_is_bootdevice(&pdev->dev))
        mmc_flush_detect_work(host->mmc);

    snprintf(boot_marker, sizeof(boot_marker),
            "M - DRIVER %s Ready", mmc_hostname(host->mmc));
    place_marker(boot_marker);

    /* Successful initialization */
    goto out;

remove_max_bus_bw_file:
    device_remove_file(&pdev->dev, &msm_host->msm_bus_vote.max_bus_bw);
remove_host:
    dead = (readl_relaxed(host->ioaddr + SDHCI_INT_STATUS) == 0xffffffff);
    pm_runtime_disable(&pdev->dev);
    sdhci_remove_host(host, dead);
vreg_deinit:
    sdhci_msm_vreg_init(&pdev->dev, msm_host->pdata, false);
bus_unregister:
    if (msm_host->msm_bus_vote.client_handle)
        sdhci_msm_bus_cancel_work_and_set_vote(host, 0);
    sdhci_msm_bus_unregister(msm_host);
sleep_clk_disable:
    if (!IS_ERR(msm_host->sleep_clk))
        clk_disable_unprepare(msm_host->sleep_clk);
ff_clk_disable:
    if (!IS_ERR(msm_host->ff_clk))
        clk_disable_unprepare(msm_host->ff_clk);
clk_disable:
    if (!IS_ERR(msm_host->clk))
        clk_disable_unprepare(msm_host->clk);
pclk_disable:
    if (!IS_ERR(msm_host->pclk))
        clk_disable_unprepare(msm_host->pclk);
bus_clk_disable:
    if (!IS_ERR_OR_NULL(msm_host->bus_clk))
        clk_disable_unprepare(msm_host->bus_clk);
pltfm_free:
    sdhci_pltfm_free(pdev);
out_host_free:
    devm_kfree(&pdev->dev, msm_host);
out:
    pr_debug("%s: Exit %s\n", dev_name(&pdev->dev), __func__);
    return ret;
}
```



### 3、CORE