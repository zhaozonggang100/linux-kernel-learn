```c
file：drivers/scsi/ufs/ufshcd.c

11083 /**
11084  * ufshcd_init - Driver initialization routine
11085  * @hba: per-adapter instance
11086  * @mmio_base: base register address
11087  * @irq: Interrupt line of device
11088  * Returns 0 on success, non-zero value on failure
11089  */
11090 int ufshcd_init(struct ufs_hba *hba, void __iomem *mmio_base, unsigned int irq)
11091 {
11092     int err;
11093     struct Scsi_Host *host = hba->host;
11094     struct device *dev = hba->dev;
11095
11096     if (!mmio_base) {
11097         dev_err(hba->dev,
11098         "Invalid memory reference for mmio_base is NULL\n");
11099         err = -ENODEV;
11100         goto out_error;
11101     }
11102
11103     hba->mmio_base = mmio_base;
11104     hba->irq = irq;
11105
11106     /* Set descriptor lengths to specification defaults */
11107     ufshcd_def_desc_sizes(hba);
11108
    	  // 电压时钟的初始化
11109     err = ufshcd_hba_init(hba);
11110     if (err)
11111         goto out_error;
11112
    	  // 通过ufshcd_readl读取ufs的capabilities
11113     /* Read capabilities registers */
11114     ufshcd_hba_capabilities(hba);
11115
11116     /* Get UFS version supported by the controller */
11117     hba->ufs_version = ufshcd_get_ufs_version(hba);
11118
11119     /* print error message if ufs_version is not valid */
11120     if ((hba->ufs_version != UFSHCI_VERSION_10) &&
11121         (hba->ufs_version != UFSHCI_VERSION_11) &&
11122         (hba->ufs_version != UFSHCI_VERSION_20) &&
11123         (hba->ufs_version != UFSHCI_VERSION_21) &&
11124         (hba->ufs_version != UFSHCI_VERSION_30))			
11125         dev_warn(hba->dev, "invalid UFS version 0x%x\n",
11126             hba->ufs_version);	// 3.0
11127
11128     /* Get Interrupt bit mask per version */
11129     hba->intr_mask = ufshcd_get_intr_mask(hba);
11130
11131     /* Enable debug prints */
11132     hba->ufshcd_dbg_print = DEFAULT_UFSHCD_DBG_PRINT_EN;
11133
    	  // 将ufshci的dma能力通知到内核
11134     err = ufshcd_set_dma_mask(hba);
11135     if (err) {
11136         dev_err(hba->dev, "set dma mask failed\n");
11137         goto out_disable;
11138     }
11139
11140     /* Allocate memory for host memory space */
    	  // 给ufshci的hba结构中的ucdl_dma_addr、utrdl_dma_addr、utmrdl_base_addr、lrb分配地址空间
11141     err = ufshcd_memory_alloc(hba);
11142     if (err) {
11143         dev_err(hba->dev, "Memory allocation failed\n");
11144         goto out_disable;
11145     }
11146
11147     /* Configure LRB */
11148     ufshcd_host_memory_configure(hba);
11149
    	  // ufs scsi host的相关参数初始化
11150     host->can_queue = hba->nutrs;
11151     host->cmd_per_lun = hba->nutrs;
11152     host->max_id = UFSHCD_MAX_ID;
11153     host->max_lun = UFS_MAX_LUNS;
11154     host->max_channel = UFSHCD_MAX_CHANNEL;
11155     host->unique_id = host->host_no;
11156     host->max_cmd_len = MAX_CDB_SIZE;
11157     host->set_dbd_for_caching = 1;
11158
11159     hba->max_pwr_info.is_valid = false;
11160
11161     /* Initailize wait queue for task management */
11162     init_waitqueue_head(&hba->tm_wq);
11163     init_waitqueue_head(&hba->tm_tag_wq);
11164
11165     /* Initialize work queues */
11166     INIT_WORK(&hba->eh_work, ufshcd_err_handler);
11167     INIT_WORK(&hba->eeh_work, ufshcd_exception_event_handler);
11168     INIT_WORK(&hba->card_detect_work, ufshcd_card_detect_handler);
11169     INIT_WORK(&hba->rls_work, ufshcd_rls_handler);
11170
11171     /* Initialize UIC command mutex */
11172     mutex_init(&hba->uic_cmd_mutex);
11173
11174     /* Initialize mutex for device management commands */
11175     mutex_init(&hba->dev_cmd.lock);
11176
11177     init_rwsem(&hba->lock);
11178
11179     /* Initialize device management tag acquire wait queue */
11180     init_waitqueue_head(&hba->dev_cmd.tag_wq);
11181
    	  // 初始化时钟
11182     ufshcd_init_clk_gating(hba);
11183     ufshcd_init_hibern8_on_idle(hba);
11184
11185     /*
11186      * In order to avoid any spurious interrupt immediately after
11187      * registering UFS controller interrupt handler, clear any pending UFS
11188      * interrupt status and disable all the UFS interrupts.
11189      */
11190     ufshcd_writel(hba, ufshcd_readl(hba, REG_INTERRUPT_STATUS),
11191               REG_INTERRUPT_STATUS);
11192     ufshcd_writel(hba, 0, REG_INTERRUPT_ENABLE);
11193     /*
11194      * Make sure that UFS interrupts are disabled and any pending interrupt
11195      * status is cleared before registering UFS interrupt handler.
11196      */
11197     mb();
11198
11199     /* IRQ registration */
    	  // 申请ufshci中断
11200     err = devm_request_irq(dev, irq, ufshcd_intr, IRQF_SHARED,
11201                 dev_name(dev), hba);
11202     if (err) {
11203         dev_err(hba->dev, "request irq failed\n");
11204         goto exit_gating;
11205     } else {
11206         hba->is_irq_enabled = true;
11207     }
11208
    	  // 将ufshci添加到scsi
11209     err = scsi_add_host(host, hba->dev);
11210     if (err) {
11211         dev_err(hba->dev, "scsi_add_host failed\n");
11212         goto exit_gating;
11213     }
11214
11215     /* Reset controller to power on reset (POR) state */
11216     ufshcd_vops_full_reset(hba);
11217
11218     /* reset connected UFS device */
11219     err = ufshcd_reset_device(hba);
11220     if (err)
11221         dev_warn(hba->dev, "%s: device reset failed. err %d\n",
11222              __func__, err);
11223
11224     if (hba->force_g4)
11225         hba->reinit_g4_rate_A = true;
11226
11227     /* Host controller enable */
11228     err = ufshcd_hba_enable(hba);
11229     if (err) {
11230         dev_err(hba->dev, "Host controller enable failed\n");
11231         ufshcd_print_host_regs(hba);
11232         ufshcd_print_host_state(hba);
11233         goto out_remove_scsi_host;
11234     }
11235
11236     if (ufshcd_is_clkscaling_supported(hba)) {
11237         char wq_name[sizeof("ufs_clkscaling_00")];
11238
11239         INIT_WORK(&hba->clk_scaling.suspend_work,
11240               ufshcd_clk_scaling_suspend_work);
11241         INIT_WORK(&hba->clk_scaling.resume_work,
11242               ufshcd_clk_scaling_resume_work);
11243
11244         snprintf(wq_name, sizeof(wq_name), "ufs_clkscaling_%d",
11245              host->host_no);
11246         hba->clk_scaling.workq = create_singlethread_workqueue(wq_name);
11247
11248         ufshcd_clkscaling_init_sysfs(hba);
11249     }
11250
11251     /*
11252      * If rpm_lvl and and spm_lvl are not already set to valid levels,
11253      * set the default power management level for UFS runtime and system
11254      * suspend. Default power saving mode selected is keeping UFS link in
11255      * Hibern8 state and UFS device in sleep state.
11256      */
11257     if (!ufshcd_is_valid_pm_lvl(hba->rpm_lvl))
11258         hba->rpm_lvl = ufs_get_desired_pm_lvl_for_dev_link_state(
11259                             UFS_SLEEP_PWR_MODE,
11260                             UIC_LINK_HIBERN8_STATE);
11261     if (!ufshcd_is_valid_pm_lvl(hba->spm_lvl))
11262         hba->spm_lvl = ufs_get_desired_pm_lvl_for_dev_link_state(
11263                             UFS_SLEEP_PWR_MODE,
11264                             UIC_LINK_HIBERN8_STATE);
11265
11266     /* Hold auto suspend until async scan completes */
11267     pm_runtime_get_sync(dev);
11268
11269     ufshcd_init_latency_hist(hba);
11270
11271     /*
11272      * We are assuming that device wasn't put in sleep/power-down
11273      * state exclusively during the boot stage before kernel.
11274      * This assumption helps avoid doing link startup twice during
11275      * ufshcd_probe_hba().
11276      */
11277     ufshcd_set_ufs_dev_active(hba);
11278
11279     ufshcd_cmd_log_init(hba);
11280
11281     async_schedule(ufshcd_async_scan, hba);
11282
11283     ufsdbg_add_debugfs(hba);
11284
11285     ufshcd_add_sysfs_nodes(hba);
11286
11287     return 0;
11288
11289 out_remove_scsi_host:
11290     scsi_remove_host(hba->host);
11291 exit_gating:
11292     ufshcd_exit_clk_gating(hba);
11293     ufshcd_exit_latency_hist(hba);
11294 out_disable:
11295     hba->is_irq_enabled = false;
11296     ufshcd_hba_exit(hba);
11297 out_error:
11298     return err;
11299 }
11300 EXPORT_SYMBOL_GPL(ufshcd_init);
```

