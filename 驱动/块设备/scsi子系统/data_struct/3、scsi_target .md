```c
// file：include/scsi/scsi_device.h

274 /*
275  * scsi_target: representation of a scsi target, for now, this is only
276  * used for single_lun devices. If no one has active IO to the target,
277  * starget_sdev_user is NULL, else it points to the active sdev.
278  */
279 struct scsi_target {
280     struct scsi_device  *starget_sdev_user; //用于目标节点一次只允许对一个lun进行IO的场合
281     struct list_head    siblings; //链接到所属主机适配器链表的连接件
282     struct list_head    devices; //这个目标节点的scsi设备链表头
283     struct device       dev;
284     struct kref     reap_ref; /* last put renders target invisible */
285     unsigned int        channel; //通道号
286     unsigned int        id; /* target id ... replace
287                      * scsi_device.id eventually */ //目标节点的ID
288     unsigned int        create:1; /* signal that it needs to be added */ //是否需要添加
289     unsigned int        single_lun:1;   /* Indicates we should only
290                          * allow I/O to one of the luns
291                          * for the device at a time. */
292     unsigned int        pdt_1f_for_no_lun:1;    /* PDT = 0x1f
293                          * means no lun present. */
294     unsigned int        no_report_luns:1;   /* Don't use
295                          * REPORT LUNS for scanning. */
296     unsigned int        expecting_lun_change:1; /* A device has reported
297                          * a 3F/0E UA, other devices on
298                          * the same target will also. */
299     /* commands actually active on LLD. */
300     atomic_t        target_busy;	// 已经派发给目标节点（低层驱动）的命令数
301     atomic_t        target_blocked;	//目标节点阻塞计数器
302
303     /*
304      * LLDs should set this in the slave_alloc host template callout.
305      * If set to zero then there is not limit.
306      */
307     unsigned int        can_queue;	//可以同时处理的命令数
308     unsigned int        max_target_blocked;
309 #define SCSI_DEFAULT_TARGET_BLOCKED 3
310
311     char            scsi_level;	// 这个scsi目标节点支持的scsi规范级别
312     enum scsi_target_state  state;
    	// 指向scsi目标节点专有数据的指针
313     void            *hostdata; /* available to low-level driver */
    	// 用于传输层
314     unsigned long       starget_data[0]; /* for the transport */
315     /* starget_data must be the last element!!!! */
316 } __attribute__((aligned(sizeof(unsigned long))));
```

