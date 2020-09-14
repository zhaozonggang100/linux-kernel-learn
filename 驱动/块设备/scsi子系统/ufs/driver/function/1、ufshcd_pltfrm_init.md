```c
2172 /**
2173  * ufs_qcom_init - bind phy with controller
2174  * @hba: host controller instance
2175  *
2176  * Binds PHY with controller and powers up PHY enabling clocks
2177  * and regulators.
2178  *
2179  * Returns -EPROBE_DEFER if binding fails, returns negative error
2180  * on phy power up failure and returns zero on success.
2181  */
/*
	该函数会在后面的ufshcd_init-->ufshcd_hba_init-->ufshcd_variant_hba_init-->ufshcd_vops_init
	1327 static inline int ufshcd_vops_init(struct ufs_hba *hba)
    1328 {
    1329     if (hba->var && hba->var->vops && hba->var->vops->init)
    1330         return hba->var->vops->init(hba);
    1331     return 0;
    1332 }
    ufs controller电压时钟初始化完成之后调用该函数建立了ufs controller和phy的连接，这样scsi低层驱动就可以通过ufs controller再通过mipi phy 和 ufs device通信了
*/
2182 static int ufs_qcom_init(struct ufs_hba *hba)
2183 {
2184     int err;
2185     struct device *dev = hba->dev;
2186     struct platform_device *pdev = to_platform_device(dev);
2187     struct ufs_qcom_host *host;
2188     struct resource *res;
2189
2190     host = devm_kzalloc(dev, sizeof(*host), GFP_KERNEL);
2191     if (!host) {
2192         err = -ENOMEM;
2193         dev_err(dev, "%s: no memory for qcom ufs host\n", __func__);
2194         goto out;
2195     }
2196
2197     /* Make a two way bind between the qcom host and the hba */
2198     host->hba = hba;
2199     spin_lock_init(&host->ice_work_lock);
2200
    	 // hba->priv = variant;
2201     ufshcd_set_variant(hba, host);
2202
2203     err = ufs_qcom_ice_get_dev(host);
2204     if (err == -EPROBE_DEFER) {
2205         /*
2206          * UFS driver might be probed before ICE driver does.
2207          * In that case we would like to return EPROBE_DEFER code
2208          * in order to delay its probing.
2209          */
2210         dev_err(dev, "%s: required ICE device not probed yet err = %d\n",
2211             __func__, err);
2212         goto out_variant_clear;
2213
2214     } else if (err == -ENODEV) {
2215         /*
2216          * ICE device is not enabled in DTS file. No need for further
2217          * initialization of ICE driver.
2218          */
2219         dev_warn(dev, "%s: ICE device is not enabled",
2220             __func__);
2221     } else if (err) {
2222         dev_err(dev, "%s: ufs_qcom_ice_get_dev failed %d\n",
2223             __func__, err);
2224         goto out_variant_clear;
2225     } else {
2226         hba->host->inlinecrypt_support = 1;
2227     }
2228
2229     host->generic_phy = devm_phy_get(dev, "ufsphy");
2230
2231     if (host->generic_phy == ERR_PTR(-EPROBE_DEFER)) {
2232         /*
2233          * UFS driver might be probed before the phy driver does.
2234          * In that case we would like to return EPROBE_DEFER code.
2235          */
2236         err = -EPROBE_DEFER;
2237         dev_warn(dev, "%s: required phy device. hasn't probed yet. err = %d\n",
2238             __func__, err);
2239         goto out_variant_clear;
2240     } else if (IS_ERR(host->generic_phy)) {
2241         err = PTR_ERR(host->generic_phy);
2242         dev_err(dev, "%s: PHY get failed %d\n", __func__, err);
2243         goto out_variant_clear;
2244     }
2245
2246     err = ufs_qcom_pm_qos_init(host);
2247     if (err)
2248         dev_info(dev, "%s: PM QoS will be disabled\n", __func__);
2249
2250     /* restore the secure configuration */
2251     ufs_qcom_update_sec_cfg(hba, true);
2252
2253     err = ufs_qcom_bus_register(host);
2254     if (err)
2255         goto out_variant_clear;
2256
2257     ufs_qcom_get_controller_revision(hba, &host->hw_ver.major,
2258         &host->hw_ver.minor, &host->hw_ver.step);
2259
2260     /*
2261      * for newer controllers, device reference clock control bit has
2262      * moved inside UFS controller register address space itself.
2263      */
2264     if (host->hw_ver.major >= 0x02) {
2265         host->dev_ref_clk_ctrl_mmio = hba->mmio_base + REG_UFS_CFG1;
2266         host->dev_ref_clk_en_mask = BIT(26);
2267     } else {
2268         /* "dev_ref_clk_ctrl_mem" is optional resource */
2269         res = platform_get_resource(pdev, IORESOURCE_MEM, 1);
2270         if (res) {
2271             host->dev_ref_clk_ctrl_mmio =
2272                     devm_ioremap_resource(dev, res);
2273             if (IS_ERR(host->dev_ref_clk_ctrl_mmio)) {
2274                 dev_warn(dev,
2275                     "%s: could not map dev_ref_clk_ctrl_mmio, err %ld\n",
2276                     __func__,
2277                     PTR_ERR(host->dev_ref_clk_ctrl_mmio));
2278                 host->dev_ref_clk_ctrl_mmio = NULL;
2279             }
2280             host->dev_ref_clk_en_mask = BIT(5);
2281         }
2282     }
2283
2284     /* update phy revision information before calling phy_init() */
2285     ufs_qcom_phy_save_controller_version(host->generic_phy,
2286         host->hw_ver.major, host->hw_ver.minor, host->hw_ver.step);
2287
2288     err = ufs_qcom_parse_reg_info(host, "qcom,vddp-ref-clk",
2289                       &host->vddp_ref_clk);
2290     phy_init(host->generic_phy);
2291
2292     if (host->vddp_ref_clk) {
2293         err = ufs_qcom_enable_vreg(dev, host->vddp_ref_clk);
2294         if (err) {
2295             dev_err(dev, "%s: failed enabling ref clk supply: %d\n",
2296                 __func__, err);
2297             goto out_unregister_bus;
2298         }
2299     }
2300
2301     err = ufs_qcom_init_lane_clks(host);
2302     if (err)
2303         goto out_disable_vddp;
2304
2305     ufs_qcom_parse_lpm(host);
2306     if (host->disable_lpm)
2307         pm_runtime_forbid(host->hba->dev);
2308     ufs_qcom_set_caps(hba);
2309     ufs_qcom_advertise_quirks(hba);
2310
2311     ufs_qcom_set_bus_vote(hba, true);
2312     ufs_qcom_setup_clocks(hba, true, POST_CHANGE);
2313
2314     host->dbg_print_en |= UFS_QCOM_DEFAULT_DBG_PRINT_EN;
2315     ufs_qcom_get_default_testbus_cfg(host);
2316     err = ufs_qcom_testbus_config(host);
2317     if (err) {
2318         dev_warn(dev, "%s: failed to configure the testbus %d\n",
2319                 __func__, err);
2320         err = 0;
2321     }
2322
2323     ufs_qcom_save_host_ptr(hba);
2324
2325     goto out;
2326
2327 out_disable_vddp:
2328     if (host->vddp_ref_clk)
2329         ufs_qcom_disable_vreg(dev, host->vddp_ref_clk);
2330 out_unregister_bus:
2331     phy_exit(host->generic_phy);
2332     msm_bus_scale_unregister_client(host->bus_vote.client_handle);
2333 out_variant_clear:
2334     devm_kfree(dev, host);
2335     ufshcd_set_variant(hba, NULL);
2336 out:
2337     return err;
2338 }

// file：drivers/scsi/ufs/ufs-qcom.c
2791 /**
2792  * struct ufs_hba_qcom_vops - UFS QCOM specific variant operations
2793  *
2794  * The variant operations configure the necessary controller and PHY
2795  * handshake during initialization.
2796  */
2797 static struct ufs_hba_variant_ops ufs_hba_qcom_vops = {
2798     .init                   = ufs_qcom_init,
2799     .exit                   = ufs_qcom_exit,
2800     .get_ufs_hci_version    = ufs_qcom_get_ufs_hci_version,
2801     .clk_scale_notify   = ufs_qcom_clk_scale_notify,
2802     .setup_clocks           = ufs_qcom_setup_clocks,
2803     .hce_enable_notify      = ufs_qcom_hce_enable_notify,
2804     .link_startup_notify    = ufs_qcom_link_startup_notify,
2805     .pwr_change_notify  = ufs_qcom_pwr_change_notify,
2806     .apply_dev_quirks   = ufs_qcom_apply_dev_quirks,
2807     .suspend        = ufs_qcom_suspend,
2808     .resume         = ufs_qcom_resume,
2809     .full_reset     = ufs_qcom_full_reset,
2810     .update_sec_cfg     = ufs_qcom_update_sec_cfg,
2811     .get_scale_down_gear    = ufs_qcom_get_scale_down_gear,
2812     .set_bus_vote       = ufs_qcom_set_bus_vote,
2813     .dbg_register_dump  = ufs_qcom_dump_dbg_regs,
2814 #ifdef CONFIG_DEBUG_FS
2815     .add_debugfs        = ufs_qcom_dbg_add_debugfs,
2816 #endif
2817 };


file：drivers/scsi/ufs/ufshcd-platform.c

479 /**
480  * ufshcd_pltfrm_init - probe routine of the driver
481  * @pdev: pointer to Platform device handle
482  * @var: pointer to variant specific data
483  *
484  * Returns 0 on success, non-zero value on failure
485  */
486 int ufshcd_pltfrm_init(struct platform_device *pdev,
487                struct ufs_hba_variant *var)
488 {
489     struct ufs_hba *hba;
490     void __iomem *mmio_base;
491     struct resource *mem_res;
492     int irq, err;
493     struct device *dev = &pdev->dev;
494
    	// 从platform_device中获取resource
495     mem_res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    	// 从上一步返回得resource中获取ufshci寄存器的基地址
496     mmio_base = devm_ioremap_resource(dev, mem_res);
497     if (IS_ERR(mmio_base)) {
498         err = PTR_ERR(mmio_base);
499         goto out;
500     }
501
502     irq = platform_get_irq(pdev, 0);	// 获取中断
503     if (irq < 0) {
504         dev_err(dev, "IRQ resource not available\n");
505         err = -ENODEV;
506         goto out;
507     }
508
     	/*
     		通过scsi_host_alloc申请一个scsi host并赋值给hba的成员
     	*/
509     err = ufshcd_alloc_host(dev, &hba);	// alloc hba resource
510     if (err) {
511         dev_err(&pdev->dev, "Allocation failed\n");
512         goto out;
513     }
514
    	// 初始化hba变体
515     hba->var = var;
516
    	// 通过dts获取时钟
517     err = ufshcd_parse_clock_info(hba);
518     if (err) {
519         dev_err(&pdev->dev, "%s: clock parse failed %d\n",
520                 __func__, err);
521         goto dealloc_host;
522     }
    
		// 从device tree初始化电压信息，ufs的工作需要三个电源电压vcc, vccq, vccq2
523     err = ufshcd_parse_regulator_info(hba);
524     if (err) {
525         dev_err(&pdev->dev, "%s: regulator init failed %d\n",
526                 __func__, err);
527         goto dealloc_host;
528     }
529
530     err = ufshcd_parse_reset_info(hba);
531     if (err) {
532         dev_err(&pdev->dev, "%s: reset parse failed %d\n",
533                 __func__, err);
534         goto dealloc_host;
535     }
536
537     err = ufshcd_parse_pinctrl_info(hba);
538     if (err) {
539         dev_dbg(&pdev->dev, "%s: unable to parse pinctrl data %d\n",
540                 __func__, err);
541         /* let's not fail the probe */
542     }
543
544     ufshcd_parse_dev_ref_clk_freq(hba);
545     ufshcd_parse_pm_levels(hba);
546     ufshcd_parse_gear_limits(hba);
547     ufshcd_parse_cmd_timeout(hba);
548     ufshcd_parse_force_g4_flag(hba);
549     err = ufshcd_parse_extcon_info(hba);
550     if (err)
551         goto dealloc_host;
552
553     if (!dev->dma_mask)
554         dev->dma_mask = &dev->coherent_dma_mask;
555
    	// 获取每个方向lane的个数
556     ufshcd_init_lanes_per_dir(hba);
557
558     err = ufshcd_init(hba, mmio_base, irq);
559     if (err) {
560         dev_err(dev, "Initialization failed\n");
561         goto dealloc_host;
562     }
563
564     platform_set_drvdata(pdev, hba);
565
566     pm_runtime_set_active(&pdev->dev);
567     pm_runtime_enable(&pdev->dev);
568
569     return 0;
570 dealloc_host:
571     ufshcd_dealloc_host(hba);
572 out:
573     return err;
574 }
575 EXPORT_SYMBOL_GPL(ufshcd_pltfrm_init);
```



```c
file ：drivers/scsi/ufs/ufshcd.c
    
11045 /**
11046  * ufshcd_alloc_host - allocate Host Bus Adapter (HBA)
11047  * @dev: pointer to device handle
11048  * @hba_handle: driver private handle
11049  * Returns 0 on success, non-zero value on failure
11050  */
11051 int ufshcd_alloc_host(struct device *dev, struct ufs_hba **hba_handle)
11052 {
11053     struct Scsi_Host *host;
11054     struct ufs_hba *hba;
11055     int err = 0;
11056
11057     if (!dev) {
11058         dev_err(dev,
11059         "Invalid memory reference for dev is NULL\n");
11060         err = -ENODEV;
11061         goto out_error;
11062     }
11063
    	  // 使用scsi_host_template模板分配一个scsi host
11064     host = scsi_host_alloc(&ufshcd_driver_template,
11065                 sizeof(struct ufs_hba));
11066     if (!host) {
11067         dev_err(dev, "scsi_host_alloc failed\n");
11068         err = -ENOMEM;
11069         goto out_error;
11070     }
11071     hba = shost_priv(host);
11072     hba->host = host;
11073     hba->dev = dev;
11074     *hba_handle = hba;
11075
11076     INIT_LIST_HEAD(&hba->clk_list_head);
11077
11078 out_error:
11079     return err;
11080 }
11081 EXPORT_SYMBOL(ufshcd_alloc_host);

 9390 static struct scsi_host_template ufshcd_driver_template = {
 9391     .module         = THIS_MODULE,
 9392     .name           = UFSHCD,
 9393     .proc_name      = UFSHCD,
 9394     .queuecommand       = ufshcd_queuecommand,
 9395     .slave_alloc        = ufshcd_slave_alloc,
 9396     .slave_configure    = ufshcd_slave_configure,
 9397     .slave_destroy      = ufshcd_slave_destroy,
 9398     .change_queue_depth = ufshcd_change_queue_depth,
 9399     .eh_abort_handler   = ufshcd_abort,
 9400     .eh_device_reset_handler = ufshcd_eh_device_reset_handler,
 9401     .eh_host_reset_handler   = ufshcd_eh_host_reset_handler,
 9402     .eh_timed_out       = ufshcd_eh_timed_out,
 9403     .ioctl          = ufshcd_ioctl,
 9404 #ifdef CONFIG_COMPAT
 9405     .compat_ioctl       = ufshcd_ioctl,
 9406 #endif
 9407     .this_id        = -1,
 9408     .sg_tablesize       = SG_ALL,
 9409     .cmd_per_lun        = UFSHCD_CMD_PER_LUN,
 9410     .can_queue      = UFSHCD_CAN_QUEUE,
 9411     .max_host_blocked   = 1,
 9412     .track_queue_depth  = 1,
 9413 };
```

