```c
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

