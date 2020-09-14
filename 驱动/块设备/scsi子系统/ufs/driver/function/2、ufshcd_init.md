```c
// file：drivers/scsi/ufs/ufshcd.c

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
    	  // 前面从dts获取了各种ufs controller的信息，这里电压时钟的初始化，并bind mipi-phy，接着从utp以下的功能就打通了
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
11153     host->max_lun = UFS_MAX_LUNS;		// 0xc100 + 0x7F
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
    	  /*
    	  		使能ufs controller，并且把 Host Controller Enable bit置1
    	  		到这一步，ufs的m-phy、uni-pro,ufs device已经初始化完成了，并且ufs m-phy进入休眠状态
    	  */
11228     err = ufshcd_hba_enable(hba);
11229     if (err) {
11230         dev_err(hba->dev, "Host controller enable failed\n");
11231         ufshcd_print_host_regs(hba);
11232         ufshcd_print_host_state(hba);
11233         goto out_remove_scsi_host;
11234     }
11235
    	  // 检测ufs controller是否支持动态时钟调整，支持的话创建对应的work和sys节点，测试系统中存在
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
    	  // 坚持auto suspend状态直到异步扫描完成
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
    	  // 异步
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



```c
// file：drivers/scsi/ufs/ufshcd.c

 9852 static int ufshcd_hba_init(struct ufs_hba *hba)
 9853 {
 9854     int err;
 9855
 9856     /*
 9857      * Handle host controller power separately from the UFS device power
 9858      * rails as it will help controlling the UFS host controller power
 9859      * collapse easily which is different than UFS device power collapse.
 9860      * Also, enable the host controller power before we go ahead with rest
 9861      * of the initialization here.
 9862      */
 9863     err = ufshcd_init_hba_vreg(hba);
 9864     if (err)
 9865         goto out;
 9866
 9867     err = ufshcd_setup_hba_vreg(hba, true);
 9868     if (err)
 9869         goto out;
 9870
 9871     err = ufshcd_init_clocks(hba);
 9872     if (err)
 9873         goto out_disable_hba_vreg;
 9874
 9875     err = ufshcd_enable_clocks(hba);
 9876     if (err)
 9877         goto out_disable_hba_vreg;
 9878
 9879     err = ufshcd_init_vreg(hba);
 9880     if (err)
 9881         goto out_disable_clks;
 9882
 9883     err = ufshcd_setup_vreg(hba, true);
 9884     if (err)
 9885         goto out_disable_clks;
 9886
 9887     err = ufshcd_variant_hba_init(hba);
 9888     if (err)
 9889         goto out_disable_vreg;
 9890
 9891     hba->is_powered = true;
 9892     goto out;
 9893
 9894 out_disable_vreg:
 9895     ufshcd_setup_vreg(hba, false);
 9896 out_disable_clks:
 9897     ufshcd_disable_clocks(hba, false);
 9898 out_disable_hba_vreg:
 9899     ufshcd_setup_hba_vreg(hba, false);
 9900 out:
 9901     return err;
 9902 }
```



```c
 5765 /**
 5766  * ufshcd_hba_enable - initialize the controller
 5767  * @hba: per adapter instance
 5768  *
 5769  * The controller resets itself and controller firmware initialization
 5770  * sequence kicks off. When controller is ready it will set
 5771  * the Host Controller Enable bit to 1.
 5772  *
 5773  * Returns 0 on success, non-zero value on failure
 5774  */
 5775 static int ufshcd_hba_enable(struct ufs_hba *hba)
 5776 {
 5777     int retry;
 5778
 5779     /*
 5780      * msleep of 1 and 5 used in this function might result in msleep(20),
 5781      * but it was necessary to send the UFS FPGA to reset mode during
 5782      * development and testing of this driver. msleep can be changed to
 5783      * mdelay and retry count can be reduced based on the controller.
 5784      */
     	  // 1、如果ufs controller被激活了，先写对应寄存器disable了ufs controller
 5785     if (!ufshcd_is_hba_active(hba))
 5786         /* change controller state to "reset state" */
 5787         ufshcd_hba_stop(hba, true);
 5788
 5789     /* UniPro link is disabled at this point */
     	  // #define ufshcd_set_link_off(hba) ((hba)->uic_link_state = UIC_LINK_OFF_STATE)
     	  // 2、关ufs controller到unipro的连接
 5790     ufshcd_set_link_off(hba);
 5791
     	  // 3、ufs设备闪存配置，上电，并使能lane（通道）
 5792     ufshcd_vops_hce_enable_notify(hba, PRE_CHANGE);
 5793
 5794     /* start controller initialization sequence */
     	  // 4、顺序的初始化ufs controller
 5795     ufshcd_hba_start(hba);
 5796
 5797     /*
 5798      * To initialize a UFS host controller HCE bit must be set to 1.
 5799      * During initialization the HCE bit value changes from 1->0->1.
 5800      * When the host controller completes initialization sequence
 5801      * it sets the value of HCE bit to 1. The same HCE bit is read back
 5802      * to check if the controller has completed initialization sequence.
 5803      * So without this delay the value HCE = 1, set in the previous
 5804      * instruction might be read back.
 5805      * This delay can be changed based on the controller.
 5806      */
  5807     msleep(1);
 5808
 5809     /* wait for the host controller to complete initialization */
     	  // 5、等待ufs controller初始化完成
 5810     retry = 10;
 5811     while (ufshcd_is_hba_active(hba)) {
 5812         if (retry) {
 5813             retry--;
 5814         } else {
 5815             dev_err(hba->dev,
 5816                 "Controller enable failed\n");
 5817             return -EIO;
 5818         }
 5819         msleep(5);
 5820     }
 5821
 5822     /* enable UIC related interrupts */
     	  // 6、写寄存器使能uic中断
 5823     ufshcd_enable_intr(hba, UFSHCD_UIC_MASK);
 5824	
     	  // 7、检测ufs PHY是否从disable状态切换到HIBERN8（休眠）状态，因为刚初始化完的UFS M-PHY应该处于休眠状态
 5825     ufshcd_vops_hce_enable_notify(hba, POST_CHANGE);
 5826
 5827     return 0;
 5828 }
```





```c
/*
	异步初始化hba
*/ 
9091 static void ufshcd_async_scan(void *data, async_cookie_t cookie)
 9092 {
 9093     struct ufs_hba *hba = (struct ufs_hba *)data;
 9094
 9095     /*
 9096      * Don't allow clock gating and hibern8 enter for faster device
 9097      * detection.
 9098      */
 9099     ufshcd_hold_all(hba);
    	  // hba（host bus adapter）开始探测设备
 9100     ufshcd_probe_hba(hba);
 9101     ufshcd_release_all(hba);
 9102
 9103     ufshcd_extcon_register(hba);
 9104 }
```





```c
 8772 /**
 8773  * ufshcd_probe_hba - probe hba to detect device and initialize
 8774  * @hba: per-adapter instance
 8775  *
 8776  * Execute link-startup and verify device initialization
 8777  */
 8778 static int ufshcd_probe_hba(struct ufs_hba *hba)
 8779 {
 8780     struct ufs_dev_desc card = {0};
 8781     int ret;
 8782     ktime_t start = ktime_get();
 8783
 8784 reinit:
     	  // 1、初始化unipro（ufs数据链路层）
 8785     ret = ufshcd_link_startup(hba);
 8786     if (ret)
 8787         goto out;
 8788
 8789     /* Debug counters initialization */
 8790     ufshcd_clear_dbg_ufs_stats(hba);
 8791     /* set the default level for urgent bkops */
 8792     hba->urgent_bkops_lvl = BKOPS_STATUS_PERF_IMPACT;
 8793     hba->is_urgent_bkops_lvl_checked = false;
 8794
 8795     /* UniPro link is active now */
     	  // 更改目前的ufs状态信息hba->uic_link_state为UIC_LINK_ACTIVE_STATE
 8796     ufshcd_set_link_active(hba);
 8797
     	  // 发送nop_out指令，确认连接状态是否正确
 8798     ret = ufshcd_verify_dev_init(hba);
 8799     if (ret)
 8800         goto out;
 8801
     	  // 发送set_flag指令及read_flag，确认初始化状态是否正确
 8802     ret = ufshcd_complete_dev_init(hba);
 8803     if (ret)
 8804         goto out;
 8805
 8806     /* clear any previous UFS device information */
 8807     memset(&hba->dev_info, 0, sizeof(hba->dev_info));
 8808
 8809     /* cache important parameters from device descriptor for later use */
 8810     ret = ufs_read_device_desc_data(hba);
 8811     if (ret)
 8812         goto out;
 8813
 8814     /* Init check for device descriptor sizes */
 8815     ufshcd_init_desc_sizes(hba);
 8816
     	  // 读取ufs 设备描述符
 8817     ret = ufs_get_device_desc(hba, &card);
  8818     if (ret) {
 8819         dev_err(hba->dev, "%s: Failed getting device info. err = %d\n",
 8820             __func__, ret);
 8821         goto out;
 8822     }
 8823
     	  // 正常不应该走下面流程
 8824     if (card.wspecversion >= 0x300 && !hba->reinit_g4_rate_A) {
 8825         unsigned long flags;
 8826         int err;
 8827
 8828         hba->reinit_g4_rate_A = true;
 8829
 8830         err = ufshcd_vops_full_reset(hba);
 8831         if (err)
 8832             dev_warn(hba->dev, "%s: full reset returned %d\n",
 8833                  __func__, err);
 8834
 8835         err = ufshcd_reset_device(hba);
 8836         if (err)
 8837             dev_warn(hba->dev, "%s: device reset failed. err %d\n",
 8838                  __func__, err);
 8839
 8840         /* Reset the host controller */
 8841         spin_lock_irqsave(hba->host->host_lock, flags);
 8842         ufshcd_hba_stop(hba, false);
 8843         spin_unlock_irqrestore(hba->host->host_lock, flags);
 8844
 8845         err = ufshcd_hba_enable(hba);
 8846         if (err)
 8847             goto out;
 8848
 8849         goto reinit;
 8850     }
 8851
     	  // 获取device_descriptor、Manufacturer Name String Descriptor并初始化hba->dev_quirks
 8852     ufs_fixup_device_setup(hba, &card);
     	  // 根据unipro的版本设置tune ufs协议层的参数
 8853     ufshcd_tune_unipro_params(hba);
 8854
 8855     ufshcd_apply_pm_quirks(hba);
 8856     if (card.wspecversion < 0x300) {
 8857         ret = ufshcd_set_vccq_rail_unused(hba,
 8858             (hba->dev_info.quirks & UFS_DEVICE_NO_VCCQ) ?
 8859             true : false);
 8860         if (ret)
 8861             goto out;
 8862     }
 8863
 8864     /* UFS device is also active now */
 8865     ufshcd_set_ufs_dev_active(hba);
  8866    ufshcd_force_reset_auto_bkops(hba);
 8867
 8868     if (ufshcd_get_max_pwr_mode(hba)) {
 8869         dev_err(hba->dev,
 8870             "%s: Failed getting max supported power mode\n",
 8871             __func__);
 8872     } else {
 8873         ufshcd_get_dev_ref_clk_gating_wait(hba, &card);
 8874
 8875         /*
 8876          * Set the right value to bRefClkFreq before attempting to
 8877          * switch to HS gears.
 8878          */
 8879         ufshcd_set_dev_ref_clk(hba);
 8880         ret = ufshcd_config_pwr_mode(hba, &hba->max_pwr_info.info);
 8881         if (ret) {
 8882             dev_err(hba->dev, "%s: Failed setting power mode, err = %d\n",
 8883                     __func__, ret);
 8884             goto out;
 8885         }
 8886     }
 8887
 8888     /*
 8889      * bActiveICCLevel is volatile for UFS device (as per latest v2.1 spec)
 8890      * and for removable UFS card as well, hence always set the parameter.
 8891      * Note: Error handler may issue the device reset hence resetting
 8892      *       bActiveICCLevel as well so it is always safe to set this here.
 8893      */
 8894     ufshcd_set_active_icc_lvl(hba);
 8895
 8896     /* set the state as operational after switching to desired gear */
 8897     hba->ufshcd_state = UFSHCD_STATE_OPERATIONAL;
 8898
 8899     /*
 8900      * If we are in error handling context or in power management callbacks
 8901      * context, no need to scan the host
 8902      */
     	  // 如果正常的话，就会进入ufs的scsi扫描阶段
 8903     if (!ufshcd_eh_in_progress(hba) && !hba->pm_op_in_progress) {
 8904         bool flag;
 8905
 8906         if (!ufshcd_query_flag_retry(hba, UPIU_QUERY_OPCODE_READ_FLAG,
 8907                 QUERY_FLAG_IDN_PWR_ON_WPE, &flag))
 8908             hba->dev_info.f_power_on_wp_en = flag;
 8909
 8910         /* Add required well known logical units to scsi mid layer */
 8911         if (ufshcd_scsi_add_wlus(hba))
 8912             goto out;
  8913
 8914         /* lower VCC voltage level */
 8915         ufshcd_set_low_vcc_level(hba, &card);
 8916
 8917         /* Initialize devfreq after UFS device is detected */
 8918         if (ufshcd_is_clkscaling_supported(hba)) {
 8919             memcpy(&hba->clk_scaling.saved_pwr_info.info,
 8920                 &hba->pwr_info,
 8921                 sizeof(struct ufs_pa_layer_attr));
 8922             hba->clk_scaling.saved_pwr_info.is_valid = true;
 8923             hba->clk_scaling.is_scaled_up = true;
 8924             if (!hba->devfreq) {
 8925                 ret = ufshcd_devfreq_init(hba);
 8926                 if (ret)
 8927                     goto out;
 8928             }
 8929             hba->clk_scaling.is_allowed = true;
 8930         }
 8931
 8932         scsi_scan_host(hba->host);
 8933         pm_runtime_put_sync(hba->dev);
 8934     }
 8935
 8936     /*
 8937      * Enable auto hibern8 if supported, after full host and
 8938      * device initialization.
 8939      */
 8940     if (ufshcd_is_auto_hibern8_supported(hba) &&
 8941         hba->hibern8_on_idle.is_enabled)
 8942         ufshcd_set_auto_hibern8_timer(hba,
 8943                       hba->hibern8_on_idle.delay_ms);
 8944 out:
 8945     if (ret) {
 8946         ufshcd_set_ufs_dev_poweroff(hba);
 8947         ufshcd_set_link_off(hba);
 8948         if (hba->extcon) {
 8949             if (!ufshcd_is_card_online(hba))
 8950                 ufsdbg_clr_err_state(hba);
 8951             ufshcd_set_card_offline(hba);
 8952         }
 8953     } else if (hba->extcon) {
 8954         ufshcd_set_card_online(hba);
 8955     }
 8956
 8957     /*
 8958      * If we failed to initialize the device or the device is not
 8959      * present, turn off the power/clocks etc.
 8960      */
 8961     if (ret && !ufshcd_eh_in_progress(hba) && !hba->pm_op_in_progress)
 8962         pm_runtime_put_sync(hba->dev);
 8963
 8964     trace_ufshcd_init(dev_name(hba->dev), ret,
 8965         ktime_to_us(ktime_sub(ktime_get(), start)),
 8966         hba->curr_dev_pwr_mode, hba->uic_link_state);
 8967     return ret;
 8968 }
```

