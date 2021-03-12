```c
1835 /**
1836  * scsi_scan_host - scan the given adapter
1837  * @shost:  adapter to scan
1838  **/
1839 void scsi_scan_host(struct Scsi_Host *shost)
1840 {
1841     struct async_scan_data *data;
1842
    	 /*
    	 	测试系统config中定义了CONFIG_SCSI_SCAN_ASYNC=y，所以scsi使用的是异步扫描
    	 */
1843     if (strncmp(scsi_scan_type, "none", 4) == 0 ||
1844         strncmp(scsi_scan_type, "manual", 6) == 0)
1845         return;
1846     if (scsi_autopm_get_host(shost) < 0)
1847         return;
1848
    	 // 为异步扫描作准备，同步扫描是不需要作准备的，同步的时候会返回NULL，执行if里面的逻辑
1849     data = scsi_prep_async_scan(shost);
1850     if (!data) {
1851         do_scsi_scan_host(shost);
1852         scsi_autopm_put_host(shost);
1853         return;
1854     }
1855
1856     /* register with the async subsystem so wait_for_device_probe()
1857      * will flush this work
1858      */
    	 // 执行异步扫描
1859     async_schedule(do_scan_async, data);
1860
1861     /* scsi_autopm_put_host(shost) is called in scsi_finish_async_scan() */
1862 }
1863 EXPORT_SYMBOL(scsi_scan_host);
```



```c
/*
	scsi中间层同步扫描scsi host连接的设备以及lu时调用
*/
1811 static void do_scsi_scan_host(struct Scsi_Host *shost)
1812 {
1813     if (shost->hostt->scan_finished) {
1814         unsigned long start = jiffies;
1815         if (shost->hostt->scan_start)
1816             shost->hostt->scan_start(shost);
1817
1818         while (!shost->hostt->scan_finished(shost, jiffies - start))
1819             msleep(10);
1820     } else {
1821         scsi_scan_host_selected(shost, SCAN_WILD_CARD, SCAN_WILD_CARD,
1822                 SCAN_WILD_CARD, 0);
1823     }
1824 }
```



```c
/*
	开始异步扫描
*/
1826 static void do_scan_async(void *_data, async_cookie_t c)
1827 {
1828     struct async_scan_data *data = _data;
1829     struct Scsi_Host *shost = data->shost;
1830
1831     do_scsi_scan_host(shost);
1832     scsi_finish_async_scan(data);
1833 }

1811 static void do_scsi_scan_host(struct Scsi_Host *shost)
1812 {
1813     if (shost->hostt->scan_finished) {
1814         unsigned long start = jiffies;
1815         if (shost->hostt->scan_start)
1816             shost->hostt->scan_start(shost);
1817
1818         while (!shost->hostt->scan_finished(shost, jiffies - start))
1819             msleep(10);
1820     } else {
    		 // here，这里扫描所有可能的逻辑设备
            /*
            * Special value for scanning to specify scanning or rescanning of all
            * possible channels, (target) ids, or luns on a given shost.
            * #define SCAN_WILD_CARD  ~0
            */
1821         scsi_scan_host_selected(shost, SCAN_WILD_CARD, SCAN_WILD_CARD,
1822                 SCAN_WILD_CARD, 0);
1823     }
1824 }

1662 int scsi_scan_host_selected(struct Scsi_Host *shost, unsigned int channel,
1663                 unsigned int id, u64 lun,
1664                 enum scsi_scan_mode rescan)
1665 {
1666     SCSI_LOG_SCAN_BUS(3, shost_printk (KERN_INFO, shost,
1667         "%s: <%u:%u:%llu>\n",
1668         __func__, channel, id, lun));
1669
1670     if (((channel != SCAN_WILD_CARD) && (channel > shost->max_channel)) ||
1671         ((id != SCAN_WILD_CARD) && (id >= shost->max_id)) ||
1672         ((lun != SCAN_WILD_CARD) && (lun >= shost->max_lun)))
1673         return -EINVAL;
1674
1675     mutex_lock(&shost->scan_mutex);
1676     if (!shost->async_scan)
1677         scsi_complete_async_scans();
1678
1679     if (scsi_host_scan_allowed(shost) && scsi_autopm_get_host(shost) == 0) {
1680         if (channel == SCAN_WILD_CARD)
1681             for (channel = 0; channel <= shost->max_channel;
1682                  channel++)
    				 // 扫描通道
1683                 scsi_scan_channel(shost, channel, id, lun,
1684                           rescan);
1685         else
1686             scsi_scan_channel(shost, channel, id, lun, rescan);
1687         scsi_autopm_put_host(shost);
1688     }
1689     mutex_unlock(&shost->scan_mutex);
1690
1691     return 0;
1692 }

1630 static void scsi_scan_channel(struct Scsi_Host *shost, unsigned int channel,
1631                   unsigned int id, u64 lun,
1632                   enum scsi_scan_mode rescan)
1633 {
1634     uint order_id;
1635
1636     if (id == SCAN_WILD_CARD)
1637         for (id = 0; id < shost->max_id; ++id) {
1638             /*
1639              * XXX adapter drivers when possible (FCP, iSCSI)
1640              * could modify max_id to match the current max,
1641              * not the absolute max.
1642              *
1643              * XXX add a shost id iterator, so for example,
1644              * the FC ID can be the same as a target id
1645              * without a huge overhead of sparse id's.
1646              */
1647             if (shost->reverse_ordering)
1648                 /*
1649                  * Scan from high to low id.
1650                  */
1651                 order_id = shost->max_id - id - 1;
1652             else
1653                 order_id = id;
    			 // 扫描节点
1654             __scsi_scan_target(&shost->shost_gendev, channel,
1655                     order_id, lun, rescan);
1656         }
1657     else
1658         __scsi_scan_target(&shost->shost_gendev, channel,
1659                 id, lun, rescan);
1660 }

1535 static void __scsi_scan_target(struct device *parent, unsigned int channel,
1536         unsigned int id, u64 lun, enum scsi_scan_mode rescan)
1537 {
1538     struct Scsi_Host *shost = dev_to_shost(parent);
1539     int bflags = 0;
1540     int res;
1541     struct scsi_target *starget;
1542
1543     if (shost->this_id == id)
1544         /*
1545          * Don't scan the host adapter
1546          */
1547         return;
1548
1549     starget = scsi_alloc_target(parent, channel, id);
1550     if (!starget)
1551         return;
1552     scsi_autopm_get_target(starget);
1553
1554     if (lun != SCAN_WILD_CARD) {
1555         /*
1556          * Scan for a specific host/chan/id/lun.
1557          */
1558         scsi_probe_and_add_lun(starget, lun, NULL, NULL, rescan, NULL);
1559         goto out_reap;
1560     }
1561
1562     /*
1563      * Scan LUN 0, if there is some response, scan further. Ideally, we
1564      * would not configure LUN 0 until all LUNs are scanned.
1565      */
    	 // probe并添加LU
1566     res = scsi_probe_and_add_lun(starget, 0, &bflags, NULL, rescan, NULL);
1567     if (res == SCSI_SCAN_LUN_PRESENT || res == SCSI_SCAN_TARGET_PRESENT) {
1568         if (scsi_report_lun_scan(starget, bflags, rescan) != 0)
1569             /*
1570              * The REPORT LUN did not scan the target,
1571              * do a sequential scan.
1572              */
1573             scsi_sequential_lun_scan(starget, bflags,
1574                          starget->scsi_level, rescan);
1575     }
1576
1577  out_reap:
1578     scsi_autopm_put_target(starget);
1579     /*
1580      * paired with scsi_alloc_target(): determine if the target has
1581      * any children at all and if not, nuke it
1582      */
1583     scsi_target_reap(starget);
1584
1585     put_device(&starget->dev);
1586 }

1028 /**
1029  * scsi_probe_and_add_lun - probe a LUN, if a LUN is found add it
1030  * @starget:    pointer to target device structure
1031  * @lun:    LUN of target device
1032  * @bflagsp:    store bflags here if not NULL
1033  * @sdevp:  probe the LUN corresponding to this scsi_device
1034  * @rescan:     if not equal to SCSI_SCAN_INITIAL skip some code only
1035  *              needed on first scan
1036  * @hostdata:   passed to scsi_alloc_sdev()
1037  *
1038  * Description:
1039  *     Call scsi_probe_lun, if a LUN with an attached device is found,
1040  *     allocate and set it up by calling scsi_add_lun.
1041  *
1042  * Return:
1043  *
1044  *   - SCSI_SCAN_NO_RESPONSE: could not allocate or setup a scsi_device
1045  *   - SCSI_SCAN_TARGET_PRESENT: target responded, but no device is
1046  *         attached at the LUN
1047  *   - SCSI_SCAN_LUN_PRESENT: a new scsi_device was allocated and initialized
1048  **/
1049 static int scsi_probe_and_add_lun(struct scsi_target *starget,
1050                   u64 lun, int *bflagsp,
1051                   struct scsi_device **sdevp,
1052                   enum scsi_scan_mode rescan,
1053                   void *hostdata)
1054 {
1055     struct scsi_device *sdev;
1056     unsigned char *result;
1057     int bflags, res = SCSI_SCAN_NO_RESPONSE, result_len = 256;
1058     struct Scsi_Host *shost = dev_to_shost(starget->dev.parent);
1059
1060     /*
1061      * The rescan flag is used as an optimization, the first scan of a
1062      * host adapter calls into here with rescan == 0.
1063      */
1064     sdev = scsi_device_lookup_by_target(starget, lun);
1065     if (sdev) {
1066         if (rescan != SCSI_SCAN_INITIAL || !scsi_device_created(sdev)) {
1067             SCSI_LOG_SCAN_BUS(3, sdev_printk(KERN_INFO, sdev,
1068                 "scsi scan: device exists on %s\n",
1069                 dev_name(&sdev->sdev_gendev)));
1070             if (sdevp)
1071                 *sdevp = sdev;
1072             else
1073                 scsi_device_put(sdev);
1074
1075             if (bflagsp)
1076                 *bflagsp = scsi_get_device_flags(sdev,
1077                                  sdev->vendor,
1078                                  sdev->model);
1079             return SCSI_SCAN_LUN_PRESENT;
1080         }
1081         scsi_device_put(sdev);
1082     } else
    		 // 分配一个逻辑设备结构
1083         sdev = scsi_alloc_sdev(starget, lun, hostdata);
1084     if (!sdev)
1085         goto out;
1086
1087     result = kmalloc(result_len, GFP_KERNEL |
1088             ((shost->unchecked_isa_dma) ? __GFP_DMA : 0));
1089     if (!result)
1090         goto out_free_sdev;
1091
1092     if (scsi_probe_lun(sdev, result, result_len, &bflags))
1093         goto out_free_result;
1094
1095     if (bflagsp)
1096         *bflagsp = bflags;
1097     /*
1098      * result contains valid SCSI INQUIRY data.
1099      */
1100     if ((result[0] >> 5) == 3) {
1101         /*
1102          * For a Peripheral qualifier 3 (011b), the SCSI
1103          * spec says: The device server is not capable of
1104          * supporting a physical device on this logical
1105          * unit.
1106          *
1107          * For disks, this implies that there is no
1108          * logical disk configured at sdev->lun, but there
1109          * is a target id responding.
1110          */
1111         SCSI_LOG_SCAN_BUS(2, sdev_printk(KERN_INFO, sdev, "scsi scan:"
1112                    " peripheral qualifier of 3, device not"
1113                    " added\n"))
1114         if (lun == 0) {
1115             SCSI_LOG_SCAN_BUS(1, {
1116                 unsigned char vend[9];
1117                 unsigned char mod[17];
1118
1119                 sdev_printk(KERN_INFO, sdev,
1120                     "scsi scan: consider passing scsi_mod."
1121                     "dev_flags=%s:%s:0x240 or 0x1000240\n",
1122                     scsi_inq_str(vend, result, 8, 16),
1123                     scsi_inq_str(mod, result, 16, 32));
1124             });
1125
1126         }
1127
1128         res = SCSI_SCAN_TARGET_PRESENT;
1129         goto out_free_result;
1130     }
1131
1132     /*
1133      * Some targets may set slight variations of PQ and PDT to signal
1134      * that no LUN is present, so don't add sdev in these cases.
1135      * Two specific examples are:
1136      * 1) NetApp targets: return PQ=1, PDT=0x1f
1137      * 2) USB UFI: returns PDT=0x1f, with the PQ bits being "reserved"
1138      *    in the UFI 1.0 spec (we cannot rely on reserved bits).
1139      *
1140      * References:
1141      * 1) SCSI SPC-3, pp. 145-146
1142      * PQ=1: "A peripheral device having the specified peripheral
1143      * device type is not connected to this logical unit. However, the
1144      * device server is capable of supporting the specified peripheral
1145      * device type on this logical unit."
1146      * PDT=0x1f: "Unknown or no device type"
1147      * 2) USB UFI 1.0, p. 20
1148      * PDT=00h Direct-access device (floppy)
1149      * PDT=1Fh none (no FDD connected to the requested logical unit)
1150      */
1151     if (((result[0] >> 5) == 1 || starget->pdt_1f_for_no_lun) &&
1152         (result[0] & 0x1f) == 0x1f &&
1153         !scsi_is_wlun(lun)) {
1154         SCSI_LOG_SCAN_BUS(3, sdev_printk(KERN_INFO, sdev,
1155                     "scsi scan: peripheral device type"
1156                     " of 31, no device added\n"));
1157         res = SCSI_SCAN_TARGET_PRESENT;
1158         goto out_free_result;
1159     }
1160
1161     res = scsi_add_lun(sdev, result, &bflags, shost->async_scan);
1162     if (res == SCSI_SCAN_LUN_PRESENT) {
1163         if (bflags & BLIST_KEY) {
1164             sdev->lockable = 0;
1165             scsi_unlock_floptical(sdev, result);
1166         }
1167     }
1168
1169  out_free_result:
1170     kfree(result);
1171  out_free_sdev:
1172     if (res == SCSI_SCAN_LUN_PRESENT) {
1173         if (sdevp) {
1174             if (scsi_device_get(sdev) == 0) {
1175                 *sdevp = sdev;
1176             } else {
1177                 __scsi_remove_device(sdev);
1178                 res = SCSI_SCAN_NO_RESPONSE;
1179             }
1180         }
1181     } else
1182         __scsi_remove_device(sdev);
1183  out:
1184     return res;
1185 }

 201 /**
 202  * scsi_alloc_sdev - allocate and setup a scsi_Device
 203  * @starget: which target to allocate a &scsi_device for
 204  * @lun: which lun
 205  * @hostdata: usually NULL and set by ->slave_alloc instead
 206  *
 207  * Description:
 208  *     Allocate, initialize for io, and return a pointer to a scsi_Device.
 209  *     Stores the @shost, @channel, @id, and @lun in the scsi_Device, and
 210  *     adds scsi_Device to the appropriate list.
 211  *
 212  * Return value:
 213  *     scsi_Device pointer, or NULL on failure.
 214  **/
 215 static struct scsi_device *scsi_alloc_sdev(struct scsi_target *starget,
 216                        u64 lun, void *hostdata)
 217 {
 218     struct scsi_device *sdev;
 219     int display_failure_msg = 1, ret;
 220     struct Scsi_Host *shost = dev_to_shost(starget->dev.parent);
 221
 222     sdev = kzalloc(sizeof(*sdev) + shost->transportt->device_size,
 223                GFP_KERNEL);
 224     if (!sdev)
 225         goto out;
 226
 227     sdev->vendor = scsi_null_device_strs;
 228     sdev->model = scsi_null_device_strs;
 229     sdev->rev = scsi_null_device_strs;
 230     sdev->host = shost;
 231     sdev->queue_ramp_up_period = SCSI_DEFAULT_RAMP_UP_PERIOD;
 232     sdev->id = starget->id;
 233     sdev->lun = lun;
 234     sdev->channel = starget->channel;
 235     mutex_init(&sdev->state_mutex);
 236     sdev->sdev_state = SDEV_CREATED;
 237     INIT_LIST_HEAD(&sdev->siblings);
 238     INIT_LIST_HEAD(&sdev->same_target_siblings);
 239     INIT_LIST_HEAD(&sdev->cmd_list);
 240     INIT_LIST_HEAD(&sdev->starved_entry);
 241     INIT_LIST_HEAD(&sdev->event_list);
 242     spin_lock_init(&sdev->list_lock);
 243     mutex_init(&sdev->inquiry_mutex);
 244     INIT_WORK(&sdev->event_work, scsi_evt_thread);
 245     INIT_WORK(&sdev->requeue_work, scsi_requeue_run_queue);
 246
 247     sdev->sdev_gendev.parent = get_device(&starget->dev);
 248     sdev->sdev_target = starget;
 249
 250     /* usually NULL and set by ->slave_alloc instead */
 251     sdev->hostdata = hostdata;
 252
 253     /* if the device needs this changing, it may do so in the
 254      * slave_configure function */
 255     sdev->max_device_blocked = SCSI_DEFAULT_DEVICE_BLOCKED;
 256
 257     /*
 258      * Some low level driver could use device->type
 259      */
 260     sdev->type = -1;
 261
 262     /*
 263      * Assume that the device will have handshaking problems,
 264      * and then fix this field later if it turns out it
 265      * doesn't
 266      */
 267     sdev->borken = 1;
 268
 269     if (shost_use_blk_mq(shost))
     		 // 分配队列
 270         sdev->request_queue = scsi_mq_alloc_queue(sdev);
 271     else
 272         sdev->request_queue = scsi_old_alloc_queue(sdev);
 273     if (!sdev->request_queue) {
 274         /* release fn is set up in scsi_sysfs_device_initialise, so
 275          * have to free and put manually here */
 276         put_device(&starget->dev);
 277         kfree(sdev);
 278         goto out;
 279     }
 280     WARN_ON_ONCE(!blk_get_queue(sdev->request_queue));
 281     sdev->request_queue->queuedata = sdev;
 282
 283     if (!shost_use_blk_mq(sdev->host)) {
 284         blk_queue_init_tags(sdev->request_queue,
 285                     sdev->host->cmd_per_lun, shost->bqt,
 286                     shost->hostt->tag_alloc_policy);
 287     }
 288     scsi_change_queue_depth(sdev, sdev->host->cmd_per_lun ?
 289                     sdev->host->cmd_per_lun : 1);
 290
 291     scsi_sysfs_device_initialize(sdev);
 292
 293     if (shost->hostt->slave_alloc) {
 294         ret = shost->hostt->slave_alloc(sdev);
 295         if (ret) {
 296             /*
 297              * if LLDD reports slave not present, don't clutter
 298              * console with alloc failure messages
 299              */
 300             if (ret == -ENXIO)
 301                 display_failure_msg = 0;
 302             goto out_device_destroy;
 303         }
 304     }
 305
 306     return sdev;
 307
 308 out_device_destroy:
 309     __scsi_remove_device(sdev);
 310 out:
 311     if (display_failure_msg)
 312         printk(ALLOC_FAILURE_MSG, __func__);
 313     return NULL;
 314 }

2271 struct request_queue *scsi_mq_alloc_queue(struct scsi_device *sdev)
2272 {
    	 // 初始化队列
2273     sdev->request_queue = blk_mq_init_queue(&sdev->host->tag_set);
2274     if (IS_ERR(sdev->request_queue))
2275         return NULL;
2276
2277     sdev->request_queue->queuedata = sdev;
2278     __scsi_init_queue(sdev->host, sdev->request_queue);
2279     return sdev->request_queue;
2280 }

2317 struct request_queue *blk_mq_init_queue(struct blk_mq_tag_set *set)
2318 {
2319     struct request_queue *uninit_q, *q;
2320
2321     uninit_q = blk_alloc_queue_node(GFP_KERNEL, set->numa_node);
2322     if (!uninit_q)
2323         return ERR_PTR(-ENOMEM);
2324
2325     q = blk_mq_init_allocated_queue(set, uninit_q);
2326     if (IS_ERR(q))
2327         blk_cleanup_queue(uninit_q);
2328
2329     return q;
2330 }
2331 EXPORT_SYMBOL(blk_mq_init_queue);

2405 struct request_queue *blk_mq_init_allocated_queue(struct blk_mq_tag_set *set,
2406                           struct request_queue *q)
2407 {
2408     /* mark the queue as mq asap */
2409     q->mq_ops = set->ops;
2410
2411     q->poll_cb = blk_stat_alloc_callback(blk_mq_poll_stats_fn,
2412                          blk_mq_poll_stats_bkt,
2413                          BLK_MQ_POLL_STATS_BKTS, q);
2414     if (!q->poll_cb)
2415         goto err_exit;
2416
2417     q->queue_ctx = alloc_percpu(struct blk_mq_ctx);
2418     if (!q->queue_ctx)
2419         goto err_exit;
2420
2421     /* init q->mq_kobj and sw queues' kobjects */
2422     blk_mq_sysfs_init(q);
2423
2424     q->queue_hw_ctx = kzalloc_node(nr_cpu_ids * sizeof(*(q->queue_hw_ctx)),
2425                         GFP_KERNEL, set->numa_node);
2426     if (!q->queue_hw_ctx)
2427         goto err_percpu;
2428
2429     q->mq_map = set->mq_map;
2430
2431     blk_mq_realloc_hw_ctxs(set, q);
2432     if (!q->nr_hw_queues)
2433         goto err_hctxs;
2434
2435     INIT_WORK(&q->timeout_work, blk_mq_timeout_work);
2436     blk_queue_rq_timeout(q, set->timeout ? set->timeout : 30 * HZ);
2437
2438     q->nr_queues = nr_cpu_ids;
2439
2440     q->queue_flags |= QUEUE_FLAG_MQ_DEFAULT;
2441
2442     if (!(set->flags & BLK_MQ_F_SG_MERGE))
2443         q->queue_flags |= 1 << QUEUE_FLAG_NO_SG_MERGE;
2444
2445     q->sg_reserved_size = INT_MAX;
2446
2447     INIT_DELAYED_WORK(&q->requeue_work, blk_mq_requeue_work);
2448     INIT_LIST_HEAD(&q->requeue_list);
2449     spin_lock_init(&q->requeue_lock);
2450
    	 // 将sdev->request_queue和blk_mq_make_reques绑定在一起，接着触发sd_probe创建设备文件
2451     blk_queue_make_request(q, blk_mq_make_request);
2452
2453     /*
2454      * Do this after blk_queue_make_request() overrides it...
2455      */
2456     q->nr_requests = set->queue_depth;
2457
2458     /*
2459      * Default to classic polling
2460      */
2461     q->poll_nsec = -1;
2462
2463     if (set->ops->complete)
2464         blk_queue_softirq_done(q, set->ops->complete);
2465
2466     blk_mq_init_cpu_queues(q, set->nr_hw_queues);
2467     blk_mq_add_queue_tag_set(set, q);
2468     blk_mq_map_swqueue(q);
2469
2470     if (!(set->flags & BLK_MQ_F_NO_SCHED)) {
2471         int ret;
2472
2473         ret = blk_mq_sched_init(q);
2474         if (ret)
2475             return ERR_PTR(ret);
2476     }
2477
2478     return q;
2479
2480 err_hctxs:
2481     kfree(q->queue_hw_ctx);
2482 err_percpu:
2483     free_percpu(q->queue_ctx);
2484 err_exit:
2485     q->mq_ops = NULL;
2486     return ERR_PTR(-ENOMEM);
2487 }
2488 EXPORT_SYMBOL(blk_mq_init_allocated_queue);
```

