```c
370 /**
371  * scsi_host_alloc - register a scsi host adapter instance.
372  * @sht:    pointer to scsi host template
373  * @privsize:   extra bytes to allocate for driver
374  *
375  * Note:
376  *  Allocate a new Scsi_Host and perform basic initialization.
377  *  The host is not published to the scsi midlayer until scsi_add_host
378  *  is called.
379  *
380  * Return value:
381  *  Pointer to a new Scsi_Host
382  **/
383 struct Scsi_Host *scsi_host_alloc(struct scsi_host_template *sht, int privsize)
384 {
385     struct Scsi_Host *shost;
386     gfp_t gfp_mask = GFP_KERNEL;
387     int index;
388
389     if (sht->unchecked_isa_dma && privsize)
390         gfp_mask |= __GFP_DMA;
391
392     shost = kzalloc(sizeof(struct Scsi_Host) + privsize, gfp_mask);
393     if (!shost)
394         return NULL;
395
396     shost->host_lock = &shost->default_lock;
397     spin_lock_init(shost->host_lock);
398     shost->shost_state = SHOST_CREATED;
399     INIT_LIST_HEAD(&shost->__devices);
400     INIT_LIST_HEAD(&shost->__targets);
401     INIT_LIST_HEAD(&shost->eh_cmd_q);
402     INIT_LIST_HEAD(&shost->starved_list);
403     init_waitqueue_head(&shost->host_wait);
404     mutex_init(&shost->scan_mutex);
405
406     index = ida_simple_get(&host_index_ida, 0, 0, GFP_KERNEL);
407     if (index < 0)
408         goto fail_kfree;
409     shost->host_no = index;	// 初始化主机适配器编号
410
411     shost->dma_channel = 0xff;
412
413     /* These three are default values which can be overridden */
414     shost->max_channel = 0;
415     shost->max_id = 8;
416     shost->max_lun = 8;
417
418     /* Give each shost a default transportt */
419     shost->transportt = &blank_transport_template;
420
421     /*
422      * All drivers right now should be able to handle 12 byte
423      * commands.  Every so often there are requests for 16 byte
424      * commands, but individual low-level drivers need to certify that
425      * they actually do something sensible with such commands.
426      */
427     shost->max_cmd_len = 12;
428     shost->hostt = sht;
429     shost->this_id = sht->this_id;
430     shost->can_queue = sht->can_queue;
431     shost->sg_tablesize = sht->sg_tablesize;
432     shost->sg_prot_tablesize = sht->sg_prot_tablesize;
433     shost->cmd_per_lun = sht->cmd_per_lun;
434     shost->unchecked_isa_dma = sht->unchecked_isa_dma;
435     shost->use_clustering = sht->use_clustering;
436     shost->no_write_same = sht->no_write_same;
437
438     if (shost_eh_deadline == -1 || !sht->eh_host_reset_handler)
439         shost->eh_deadline = -1;
440     else if ((ulong) shost_eh_deadline * HZ > INT_MAX) {
441         shost_printk(KERN_WARNING, shost,
442                  "eh_deadline %u too large, setting to %u\n",
443                  shost_eh_deadline, INT_MAX / HZ);
444         shost->eh_deadline = INT_MAX;
445     } else
446         shost->eh_deadline = shost_eh_deadline * HZ;
447
448     if (sht->supported_mode == MODE_UNKNOWN)
449         /* means we didn't set it ... default to INITIATOR */
450         shost->active_mode = MODE_INITIATOR;
451     else
452         shost->active_mode = sht->supported_mode;
453
454     if (sht->max_host_blocked)
455         shost->max_host_blocked = sht->max_host_blocked;
456     else
457         shost->max_host_blocked = SCSI_DEFAULT_HOST_BLOCKED;
458
459     /*
460      * If the driver imposes no hard sector transfer limit, start at
461      * machine infinity initially.
462      */
463     if (sht->max_sectors)
464         shost->max_sectors = sht->max_sectors;
465     else
466         shost->max_sectors = SCSI_DEFAULT_MAX_SECTORS;
467
468     /*
469      * assume a 4GB boundary, if not set
470      */
471     if (sht->dma_boundary)
472         shost->dma_boundary = sht->dma_boundary;
473     else
474         shost->dma_boundary = 0xffffffff;
475
476     shost->use_blk_mq = scsi_use_blk_mq;
477     shost->use_blk_mq = scsi_use_blk_mq || shost->hostt->force_blk_mq;
478
479     device_initialize(&shost->shost_gendev);
480     dev_set_name(&shost->shost_gendev, "host%d", shost->host_no);
481     shost->shost_gendev.bus = &scsi_bus_type;
482     shost->shost_gendev.type = &scsi_host_type;
483
484     device_initialize(&shost->shost_dev);
485     shost->shost_dev.parent = &shost->shost_gendev;
486     shost->shost_dev.class = &shost_class;
487     dev_set_name(&shost->shost_dev, "host%d", shost->host_no);
488     shost->shost_dev.groups = scsi_sysfs_shost_attr_groups;
489
490     shost->ehandler = kthread_run(scsi_error_handler, shost,
491             "scsi_eh_%d", shost->host_no);
492     if (IS_ERR(shost->ehandler)) {
493         shost_printk(KERN_WARNING, shost,
494             "error handler thread failed to spawn, error = %ld\n",
495             PTR_ERR(shost->ehandler));
496         goto fail_index_remove;
497     }
498
499     shost->tmf_work_q = alloc_workqueue("scsi_tmf_%d",
500                         WQ_UNBOUND | WQ_MEM_RECLAIM,
501                        1, shost->host_no);
502     if (!shost->tmf_work_q) {
503         shost_printk(KERN_WARNING, shost,
                         504                  "failed to create tmf workq\n");
505         goto fail_kthread;
506     }
507     scsi_proc_hostdir_add(shost->hostt);
508     return shost;
509
510  fail_kthread:
511     kthread_stop(shost->ehandler);
512  fail_index_remove:
513     ida_simple_remove(&host_index_ida, shost->host_no);
514  fail_kfree:
515     kfree(shost);
516     return NULL;
517 }
518 EXPORT_SYMBOL(scsi_host_alloc);
```

