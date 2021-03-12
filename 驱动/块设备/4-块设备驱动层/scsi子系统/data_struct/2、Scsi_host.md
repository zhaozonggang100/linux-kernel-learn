```c
// file：include/scsi/scsi_host.h

541 struct Scsi_Host {
542     /*
543      * __devices is protected by the host_lock, but you should
544      * usually use scsi_device_lookup / shost_for_each_device
545      * to access it and don't care about locking yourself.
546      * In the rare case of being in irq context you can use
547      * their __ prefixed variants with the lock held. NEVER
548      * access this list directly from a driver.
549      */
550     struct list_head    __devices;	// scsi_device（LU）列表
551     struct list_head    __targets;	// scsi_target（ufs 设备）列表
552
553     struct list_head    starved_list;
554
555     spinlock_t      default_lock;
556     spinlock_t      *host_lock;
557
558     struct mutex        scan_mutex;/* serialize scanning activity */
559
560     struct list_head    eh_cmd_q;
561     struct task_struct    * ehandler;  /* Error recovery thread. */
562     struct completion     * eh_action; /* Wait for specific actions on the
563                           host. */
564     wait_queue_head_t       host_wait;
565     struct scsi_host_template *hostt;
566     struct scsi_transport_template *transportt;
567
568     /*
569      * Area to keep a shared tag map (if needed, will be
570      * NULL if not).
571      */
572     union {
573         struct blk_queue_tag    *bqt;
574         struct blk_mq_tag_set   tag_set;
575     };
576
577     atomic_t host_busy;        /* commands actually active on low-level */
578     atomic_t host_blocked;
579
580     unsigned int host_failed;      /* commands that failed.
581                           protected by host_lock */
582     unsigned int host_eh_scheduled;    /* EH scheduled without command */
583
584     unsigned int host_no;  /* Used for IOCTL_GET_IDLUN, /proc/scsi et al. */
585
586     /* next two fields are used to bound the time spent in error handling */
587     int eh_deadline;
588     unsigned long last_reset;
589
590
591     /*
592      * These three parameters can be used to allow for wide scsi,
593      * and for host adapters that support multiple busses
594      * The last two should be set to 1 more than the actual max id
595      * or lun (e.g. 8 for SCSI parallel systems).
596      */
597     unsigned int max_channel;
598     unsigned int max_id;
599     u64 max_lun;
600
601     /*
602      * This is a unique identifier that must be assigned so that we
603      * have some way of identifying each detected host adapter properly
604      * and uniquely.  For hosts that do not support more than one card
605      * in the system at one time, this does not need to be set.  It is
606      * initialized to 0 in scsi_register.
607      */
608     unsigned int unique_id;
609
610     /*
611      * The maximum length of SCSI commands that this host can accept.
612      * Probably 12 for most host adapters, but could be 16 for others.
613      * or 260 if the driver supports variable length cdbs.
614      * For drivers that don't set this field, a value of 12 is
615      * assumed.
616      */
617     unsigned short max_cmd_len;
618
619     int this_id;
620     int can_queue;
621     short cmd_per_lun;
622     short unsigned int sg_tablesize;
623     short unsigned int sg_prot_tablesize;
624     unsigned int max_sectors;
625     unsigned long dma_boundary;
626     /*
627      * In scsi-mq mode, the number of hardware queues supported by the LLD.
628      *
629      * Note: it is assumed that each hardware queue has a queue depth of
630      * can_queue. In other words, the total queue depth per host
631      * is nr_hw_queues * can_queue.
632      */
633     unsigned nr_hw_queues;
634     /*
635      * Used to assign serial numbers to the cmds.
636      * Protected by the host lock.
637      */
638     unsigned long cmd_serial_number;
639
640     unsigned active_mode:2;
641     unsigned unchecked_isa_dma:1;
642     unsigned use_clustering:1;
643
644     /*
645      * Host has requested that no further requests come through for the
646      * time being.
647      */
648     unsigned host_self_blocked:1;
649
650     /*
651      * Host uses correct SCSI ordering not PC ordering. The bit is
652      * set for the minority of drivers whose authors actually read
653      * the spec ;).
654      */
655     unsigned reverse_ordering:1;
656
657     /* Task mgmt function in progress */
658     unsigned tmf_in_progress:1;
659
660     /* Asynchronous scan in progress */
661     unsigned async_scan:1;
662
663     /* Don't resume host in EH */
664     unsigned eh_noresume:1;
665
666     /* The controller does not support WRITE SAME */
667     unsigned no_write_same:1;
668
669     /* Inline encryption support? */
670     unsigned inlinecrypt_support:1;
671
672     unsigned use_blk_mq:1;
673     unsigned use_cmd_list:1;
674
675     /* Host responded with short (<36 bytes) INQUIRY result */
676     unsigned short_inquiry:1;
677
678     /*
679      * Set "DBD" field in mode_sense caching mode page in case it is
680      * mandatory by LLD standard.
681      */
682     unsigned set_dbd_for_caching:1;
683
684     /*
685      * Optional work queue to be utilized by the transport
686      */
687     char work_q_name[20];
688     struct workqueue_struct *work_q;
689
690     /*
691      * Task management function work queue
692      */
693     struct workqueue_struct *tmf_work_q;
694
695     /* The transport requires the LUN bits NOT to be stored in CDB[1] */
696     unsigned no_scsi2_lun_in_cdb:1;
697
698     /*
699      * Value host_blocked counts down from
700      */
701     unsigned int max_host_blocked;
702
703     /* Protection Information */
704     unsigned int prot_capabilities;
705     unsigned char prot_guard_type;
706
707     /* legacy crap */
708     unsigned long base;
709     unsigned long io_port;
710     unsigned char n_io_port;
711     unsigned char dma_channel;
712     unsigned int  irq;
713
714
715     enum scsi_host_state shost_state;
716
717     /* ldm bits */
718     struct device       shost_gendev, shost_dev;
719
720     /*
721      * List of hosts per template.
722      *
723      * This is only for use by scsi_module.c for legacy templates.
724      * For these access to it is synchronized implicitly by
725      * module_init/module_exit.
726      */
727     struct list_head sht_legacy_list;
728
729     /*
730      * Points to the transport data (if any) which is allocated
731      * separately
732      */
733     void *shost_data;
734
735     /*
736      * Points to the physical bus device we'd use to do DMA
737      * Needed just in case we have virtual hosts.
738      */
739     struct device *dma_dev;
740
741     /*
742      * We should ensure that this is aligned, both for better performance
743      * and also because some compilers (m68k) don't automatically force
744      * alignment to a long boundary.
745      */
746     unsigned long hostdata[0]  /* Used for storage of host specific stuff */
747         __attribute__ ((aligned (sizeof(unsigned long))));
748 };
```

