### 1、info

### 2、data struct

```c++
struct scsi_cmnd {
        struct scsi_request req;
        // 指向命令所属的scsi设备
        struct scsi_device *device;
        struct list_head eh_entry; /* entry for the host eh_cmd_q */
        struct delayed_work abort_work;

        struct rcu_head rcu;

        int eh_eflags;          /* Used by error handlr */

        /*
         * This is set to jiffies as it was when the command was first
         * allocated.  It is used to time how long the command has
         * been outstanding
         */
        // 命令首次分配时的滴答数，用来计算命令已经过去多长时间了
        unsigned long jiffies_at_alloc;

        // 命令重试的次数
        int retries;
        // 可允许重试的次数
        int allowed;

        unsigned char prot_op;
        unsigned char prot_type;
        unsigned char prot_flags;

        // scsi命令的长度
        unsigned short cmd_len;
        // scsi命令的数据传输方向
        enum dma_data_direction sc_data_direction;

        /* These elements define the operation we are about to perform */
        /* 指向scsi规范格式的命令字符串 */
        unsigned char *cmnd;


        /* These elements define the operation we ultimately want to perform */
        /* scsi命令的数据缓存区 */
        struct scsi_data_buffer sdb;
        /* scsi命令的保护数据缓存区 */
        struct scsi_data_buffer *prot_sdb;

        /* 如果传输的数据小于这个量则返回错误 */
        unsigned underflow;     /* Return error if less than
                                   this amount is transferred */
        /* 传输单位，应该等于硬件扇区长度 */
        unsigned transfersize;  /* How much we are guaranteed to
                                   transfer with each SCSI transfer
                                   (ie, between disconnect / 
                                   reconnects.   Probably == sector
                                   size */
        /* 指向对应的块设备驱动层请求描述符的指针 */
        struct request *request;        /* The command we are
                                           working on */
        /* scsi命令的感测数据缓存区 */
        unsigned char *sense_buffer;
                                /* obtained by REQUEST SENSE when
                                 * CHECK CONDITION is received on original
                                 * command (auto-sense). Length must be
                                 * SCSI_SENSE_BUFFERSIZE bytes. */

        /* Low-level done function - can be used by low-level driver to point
         *        to completion function.  Not used by mid/upper level code. */
        /* 被底层驱动用来指向完成函数，中间层和上层不用 */
        void (*scsi_done) (struct scsi_cmnd *);

        /*
         * The following fields can be written to by the host specific code. 
         * Everything else should be left alone. 
         */
        // 便签
        struct scsi_pointer SCp;        /* Scratchpad used by some host adapters */

        // 被底层驱动使用
        unsigned char *host_scribble;   /* The host adapter is allowed to
                                         * call scsi_malloc and get some memory
                                         * and hang it here.  The host adapter
                                         * is also expected to call scsi_free
                                         * to release this memory.  (The memory
                                         * obtained by scsi_malloc is guaranteed
                                         * to be at an address < 16Mb). */

        // 从底层驱动返回的错误码
        int result;             /* Status code from lower level driver */
        int flags;              /* Command flags */
        unsigned long state;    /* Command completion state */

        // scsi命令标签
        unsigned char tag;      /* SCSI-II queued command tag */
        unsigned int extra_len; /* length of alignment and padding */
};
```