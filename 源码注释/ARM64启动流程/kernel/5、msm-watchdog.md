参考：

https://www.freesion.com/article/6467523351/

https://blog.csdn.net/rikeyone/article/details/103356940/

### 1、概述

msm是高通的一个系列产品，高通的硬件watchdog分为：

​	non secure watchdog：kernel管理

​	secure watchdog：tz管理

### 2、no secure watchdog

- 1、初始化模块

pure_initcall(init_watchdog);

- 2、初始化函数中注册platform driver

```c
static struct platform_driver msm_watchdog_driver = { 
    .probe = msm_watchdog_probe,  // driver注册之后，platform的match函数会设备树匹配，匹配成功调用该函数
    .remove = msm_watchdog_remove,
    .driver = { 
        .name = MODULE_NAME,
        .owner = THIS_MODULE,
        .pm = &msm_watchdog_dev_pm_ops,
        .of_match_table = msm_wdog_match_table,	// 和设备树用来匹配
    },  
};
static int init_watchdog(void)
{
    return platform_driver_register(&msm_watchdog_driver);
}
```

- 3、驱动probe函数调用

```c
static int msm_watchdog_probe(struct platform_device *pdev)
{
    int ret;
    struct msm_watchdog_data *wdog_dd;
    struct md_region md_entry;
    
    if (!pdev->dev.of_node || !enable)
        return -ENODEV;
    wdog_dd = kzalloc(sizeof(struct msm_watchdog_data), GFP_KERNEL);
    if (!wdog_dd)
        return -EIO;
    ret = msm_wdog_dt_to_pdata(pdev, wdog_dd);
    if (ret) 
        goto err;
    
    wdog_data = wdog_dd;
    wdog_dd->dev = &pdev->dev; 
    platform_set_drvdata(pdev, wdog_dd);
    cpumask_clear(&wdog_dd->alive_mask);
    /*
    	创建内核线程msm_watchdog
    */
    wdog_dd->watchdog_task = kthread_create(watchdog_kthread, wdog_dd,
            "msm_watchdog");
    if (IS_ERR(wdog_dd->watchdog_task)) {
        ret = PTR_ERR(wdog_dd->watchdog_task);
        goto err;
    }
    init_watchdog_data(wdog_dd);
    
    /* Add wdog info to minidump table */
    strlcpy(md_entry.name, "KWDOGDATA", sizeof(md_entry.name));
    md_entry.virt_addr = (uintptr_t)wdog_dd;
    md_entry.phys_addr = virt_to_phys(wdog_dd);
    md_entry.size = sizeof(*wdog_dd);
    if (msm_minidump_add_region(&md_entry))
        pr_info("Failed to add Watchdog data in Minidump\n");
    
    return 0;
err:
    kzfree(wdog_dd);
    return ret;
}
```

- 4、内核线程watchdog_kthread进行喂狗

```c
static __ref int watchdog_kthread(void *arg)
{
    struct msm_watchdog_data *wdog_dd =
        (struct msm_watchdog_data *)arg;
    unsigned long delay_time = 0;
    // 设置该内核线程优先级为最大优先级-1
    struct sched_param param = {.sched_priority = MAX_RT_PRIO-1};
    int ret, cpu;

    // 设置为RT线程
    sched_setscheduler(current, SCHED_FIFO, &param);
    
    // 循环喂狗
    while (!kthread_should_stop()) {
        do {
            // 等待等待队列，在pet_task_wakeup中会定时唤醒等待队列中的线程
            ret = wait_event_interruptible(wdog_dd->pet_complete,
                        wdog_dd->timer_expired);
        } while (ret != 0);

        wdog_dd->thread_start = sched_clock();
        for_each_cpu(cpu, cpu_present_mask)
            wdog_dd->ping_start[cpu] = wdog_dd->ping_end[cpu] = 0;

        if (wdog_dd->do_ipi_ping)
            ping_other_cpus(wdog_dd);

        do {
            ret = wait_event_interruptible(wdog_dd->pet_complete,
                        wdog_dd->user_pet_complete);
        } while (ret != 0);

        wdog_dd->timer_expired = false;
        wdog_dd->user_pet_complete = !wdog_dd->user_pet_enabled;

        if (enable) {
            delay_time = msecs_to_jiffies(wdog_dd->pet_time);
            pet_watchdog(wdog_dd);
        }
        /* Check again before scheduling
         * Could have been changed on other cpu
         */
        // 重新设置定时器实现定时喂狗
        mod_timer(&wdog_dd->pet_timer, jiffies + delay_time);
    }
    return 0;
}
```

