https://blog.csdn.net/zwj0403/article/details/38409061

https://blog.csdn.net/ooonebook/article/details/55272522

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
            struct mmc_queue queue;
            struct gendisk *disk;
            unsigned int    usage; // 是否在使用
            unsigned int    read_only; // 只读属性
            unsigned int    part_type;
            unsigned int    name_idx; 
            unsigned int    reset_done;
            unsigned int    part_curr;  // 分区号
}
```

4、struct mmc_queue，mmc的card驱动对应请求队列，内部包含struct request_queue  
```
struct mmc_queue {
    struct mmc_card     *card;
    // mmc绑定task从IO调度器队列fetch request
    struct task_struct  *thread;  
    // 从IO调度器fetch的request交给该函数处理
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

- 3、 初始化流程：  

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
}
```

3、mmc_blk_alloc调用mmc_blk_alloc_req执行真正的分配  
```
struct mmc_blk_data *mmc_blk_alloc_req(struct mmc_card *card,
    struct device *parent,sector_t size,bool default_ro,
    const char *subname, int area_type) 
{
    struct mmc_blk_data *md;  // 包含gendisk的md
    md = kzalloc(sizeof(struct mmc_blk_data), GFP_KERNEL);
    md->read_only = mmc_blk_readonly(card); // 设置card只读
    md->disk = alloc_disk(perdev_minors); // 分配gendisk
    // 初始化queue，mmc_queue中包含request_queue
    ret = mmc_init_queue(&md->queue, card, NULL, subname, area_type); 
    // mmc_blk_issue_rq函数用来处理从IO调度层fetch的request
    md->queue.issue_fn = mmc_blk_issue_rq;
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
    // 初始化mmc_queue->queue（struct request_queue），也就是绑定queue和处理函数
    // 通用块设备层接受来自fs的request需要用mmc_request_fn处理
    mq->queue = blk_init_queue(mmc_request_fn, lock);
    // IO调度器会在适当的时机唤醒内核线程mmc_queue_thread，从request_queue中fetch
    // request
    mq->thread = kthread_run(mmc_queue_thread, mq, "mmcqd/%d%s",
    host->index, subname ? subname : "");
}
```

5、mmc_queue_thread内核线程在不处理来自通用块设备层的request时会进入TASK_INTERRUPTIBLE状态  
```
int mmc_queue_thread(void *d) 
{   // d就是struct mmc_queue *mq
    struct mmc_queue *mq = d;
    struct request_queue *q = mq->queue;
    scheduler_params.sched_priority = 1;  // 设置线程优先级
    // 设置调度累为sched_fifo，实时调度，优先级为1
    sched_setscheduler(current, SCHED_FIFO, &scheduler_params); 
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
    // 根据md包含的gendisk所代表的分区，在mmc_card中选择该分区的位置
    ret = mmc_blk_part_switch(card, md);
    // 
    mmc_blk_write_packing_control(mq, req);
    if (cmd_flags & REQ_DISCARD) { // 移除card

    } else if (cmd_flags & (REQ_FLUSH | REQ_BARRIER)) { // 刷新card
    	
    } else { // 正常处理读写request
    /*
    mq:struct mmc_queue
    req:struct request
    */
    ret = mmc_blk_issue_rw_rq(mq, req);
    }
}
```

7、上一步选择完mmc_crad分区，也就是gendisk对应的分区，也是读写请求对应的分区，接着开始处理请求  
```
int mmc_blk_issue_rw_rq(struct mmc_queue *mq, struct request *rqc)
{
    struct mmc_blk_data *md = mq->data;
    struct mmc_card *card = md->queue.card;
    struct mmc_blk_request *brq = &mq->mqrq_cur->brq;
    struct mmc_queue_req *mq_rq;
    struct request *req = rqc;
    struct mmc_async_req *areq;
    if (rqc) {
    	reqs = mmc_blk_prep_packed_list(mq, rqc);
    }
    do {
    	if (rqc) {
    		// 前面有校验是否4KB对齐读写
    		if (reqs >= packed_nr) {
    			mmc_blk_packed_hdr_wrq_prep(mq->mqrq_cur,card, mq);
    		} else {
             	mmc_blk_rw_rq_prep(mq->mqrq_cur, card, 0, mq);
            }     
            areq = &mq->mqrq_cur->mmc_active;
    	} else {
    		areq = NULL;
    	}
    	// mmc驱动开始处理request
    	areq = mmc_start_req(card->host, areq, (int *) &status);
    } while (ret);
    return 1;
}
```

8、MMC驱动开始使用非阻塞方式处理request，MMC host要将reuqest转换为COMMAND

```
/*
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
		1、此处就要介绍到高通的host controller了，controller是直接通过mmc物理总			线连接EMMC内部的MMC接口（cmd、data0~7、clk）
		2、mmc子系统使用core层管理card和host，在host层又抽象出sdhc标准，高通的			msm系列和三星的s3c系列都使用了该标准，高通msm系列的实现在sdhci-msm.c中
		3、sdhci抽象出host公共部分，vendor只需要实现pinctl、power等自己特有的部		分，然后实现sdhci提供的接口
		4、下面的host代表高通的符合sdhci标准的host，core层交给host处理读写数据的			api是sdhci_request，也就是host->ops->request，在host驱动初始化时会设置
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
			// 正常写数据应该走这边，这个函数才是真正的将request中的数据转换成mm命			 //	令
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
	*/
	int sg_count;
	sg_count = dma_map_sg(mmc_dev(host->mmc), data->sg, data->sg_len,
                data->flags & MMC_DATA_WRITE ?
                DMA_TO_DEVICE : DMA_FROM_DEVICE);
    return sg_count;
}                      
```
###　2、HOST

### 3、CORE