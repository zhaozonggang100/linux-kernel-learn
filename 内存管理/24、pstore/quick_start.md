参考：

https://www.cnblogs.com/gmpy/articles/13177728.html

### 1、概述

### 2、测试

参考：https://unix.stackexchange.com/questions/273352/how-to-enable-kernel-pstore

1、配置kernel

```
CONFIG_CHROMEOS_PSTORE=m
CONFIG_PSTORE=y
CONFIG_PSTORE_CONSOLE=y
# CONFIG_PSTORE_PMSG is not set
# CONFIG_PSTORE_FTRACE is not set
CONFIG_PSTORE_RAM=m
```

2、设置设备树

```c
reserved-memory {
		#address-cells = <2>;
		#size-cells = <2>;
		ranges;

		ramoops@8f000000 {
			compatible = "ramoops";
			reg = <0 0xff000000 0 0x100000>;	// 前两个32位数字代表address-cells，表示内存起始地址，后两个32位数字代表size-cells，表示地址长度，代表1M内存
			record-size = <0x4000>;		// 16k
			console-size = <0x4000>;	// 16k
		};
	};
```

3、触发pstore

```
echo 0 > /sys/module/msm_poweroff/parameters/download_mode
echo c > /proc/sysrq-trigger
```



### 3、文件系统参数

```
1、/sys/module/ramoops/parameters/	：驱动中定义的模块参数，通过设备树节点初始化的
	console_size ：console日志crash时保存到预留内存的大小
	dump_oops 
	ecc 
	ftrace_size 
	mem_address 
	mem_size 
	mem_type 
	pmsg_size 
	record_size
2、/sys/module/ramoops/parameters/
	backend：设备树节点名字
	update_ms：？？？
3、/dev/pmsg0
	当使能CONFIG_PSTORE_PMSG时会在dev下面创建该设备文件，用于用户态写入
4、/sys/fs/pstore/pmsg-ramoops-[ID]
	当使能CONFIG_PSTORE_PMSG，reboot时会在/sys/fs/pstore/生成对应的文件，文件中的内容是从设备树配置的内存中读出来的
```



### 4、从panic到pstore，触发pstore的地方还有printk，pmsg

1、panic函数

```c
kernel/panic.c
    
void panic(const char *fmt, ...)
{
	kmsg_dump(KMSG_DUMP_PANIC);
}

/*
	在user通过reboot命令warm reboot设备也会执行kmsg_dump
*/
kernel/reboot.c
214 void kernel_restart(char *cmd)
215 {
216     kernel_restart_prepare(cmd);
217     migrate_to_reboot_cpu();
218     syscore_shutdown();
219     if (!cmd)
220         pr_emerg("Restarting system\n");
221     else
222         pr_emerg("Restarting system with command '%s'\n", cmd);
223     kmsg_dump(KMSG_DUMP_RESTART);
224     machine_restart(cmd);
225 }
```

2、kmsg_dump函数遍历dump_list获取其他模块注册的kmsg_dumper，执行对应的dump函数

```c
kernel/printk.c
    
2868 void kmsg_dump(enum kmsg_dump_reason reason)
2869 {
2870     struct kmsg_dumper *dumper;
2871     unsigned long flags;
2872 
2873     if ((reason > KMSG_DUMP_OOPS) && !always_kmsg_dump)
2874         return;
2875    
2876     rcu_read_lock();
2877     list_for_each_entry_rcu(dumper, &dump_list, list) {
2878         if (dumper->max_reason && reason > dumper->max_reason)
2879             continue;
2880    
2881         /* initialize iterator with data about the stored records */
2882         dumper->active = true;
2883 
2884         raw_spin_lock_irqsave(&logbuf_lock, flags);
2885         dumper->cur_seq = clear_seq;
2886         dumper->cur_idx = clear_idx;
2887         dumper->next_seq = log_next_seq;
2888         dumper->next_idx = log_next_idx;
2889         raw_spin_unlock_irqrestore(&logbuf_lock, flags);
2890 
2891         /* invoke dumper which will iterate over records */
    		 // 这里会遍历到pstore注册的dumper
2892         dumper->dump(dumper, reason);
2893 
2894         /* reset iterator */
2895         dumper->active = false;
2896     }
2897     rcu_read_unlock();
2898 }
```

3、pstore中注册kmsg_dumper过程

```c
1、fs/pstore/ram.c中调用ramoops_init初始化pstore

607 static struct platform_driver ramoops_driver = {
608     .probe      = ramoops_probe,
609     .remove     = ramoops_remove,
610     .driver     = {
611         .name   = "ramoops",
612     },
613 };
650 static int __init ramoops_init(void)
651 {
    	// 注册device
652     ramoops_register_dummy();
    	// 注册driver，platforform总线会自动匹配调用driver的probe函数
653     return platform_driver_register(&ramoops_driver);
654 }
655 postcore_initcall(ramoops_init);

615 static void ramoops_register_dummy(void)
616 {
617     if (!mem_size)	// 判断mem_size = 0 直接退出去从设备树获取
618         return;
619 
620     pr_info("using module parameters\n");
621 
622     dummy_data = kzalloc(sizeof(*dummy_data), GFP_KERNEL);
623     if (!dummy_data) {
624         pr_info("could not allocate pdata\n");
625         return;
626     }
627 
628     dummy_data->mem_size = mem_size;
629     dummy_data->mem_address = mem_address;
630     dummy_data->mem_type = mem_type;
631     dummy_data->record_size = record_size;
632     dummy_data->console_size = ramoops_console_size;
633     dummy_data->ftrace_size = ramoops_ftrace_size;
634     dummy_data->pmsg_size = ramoops_pmsg_size;
635     dummy_data->dump_oops = dump_oops;
636     /*
637      * For backwards compatibility ramoops.ecc=1 means 16 bytes ECC
638      * (using 1 byte for ECC isn't much of use anyway).
639      */
640     dummy_data->ecc_info.ecc_size = ramoops_ecc == 1 ? 16 : ramoops_ecc;
641 
642     dummy = platform_device_register_data(NULL, "ramoops", -1,
643             dummy_data, sizeof(struct ramoops_platform_data));
644     if (IS_ERR(dummy)) {
645         pr_info("could not create platform device: %ld\n",
646             PTR_ERR(dummy));
647     }
648 }

3、调用driver的probe函数
363 static struct ramoops_context oops_cxt = {
364     .pstore = {
365         .owner  = THIS_MODULE,
366         .name   = "ramoops",
367         .open   = ramoops_pstore_open,
368         .read   = ramoops_pstore_read,
369         .write_buf  = ramoops_pstore_write_buf,
370         .erase  = ramoops_pstore_erase,
371     },
372 };

470 static int ramoops_probe(struct platform_device *pdev)
471 {
472     struct device *dev = &pdev->dev;
473     struct ramoops_platform_data *pdata = pdev->dev.platform_data;
474     struct ramoops_context *cxt = &oops_cxt;
475     size_t dump_mem_sz;
476     phys_addr_t paddr;
477     int err = -EINVAL;
478 
    	// 从设备树获取各个区域的大小 
726     if (dev_of_node(dev) && !pdata) {
727         pdata = &pdata_local;
728         memset(pdata, 0, sizeof(*pdata));
729     
730         err = ramoops_parse_dt(pdev, pdata);
731         if (err < 0)
732             goto fail_out;
733     }
479     /* Only a single ramoops area allowed at a time, so fail extra
480      * probes.
481      */
482     if (cxt->max_dump_cnt)
483         goto fail_out;
484 
485     if (!pdata->mem_size || (!pdata->record_size && !pdata->console_size &&
486             !pdata->ftrace_size && !pdata->pmsg_size)) {
487         pr_err("The memory size and the record/console size must be "
488             "non-zero\n");
489         goto fail_out;
490     }
491 
    	// 判断record_size是否为0，不为0是否是2的次数幂，不是进行四舍五入
492     if (pdata->record_size && !is_power_of_2(pdata->record_size))
493         pdata->record_size = rounddown_pow_of_two(pdata->record_size);
494     if (pdata->console_size && !is_power_of_2(pdata->console_size))
495         pdata->console_size = rounddown_pow_of_two(pdata->console_size);
496     if (pdata->ftrace_size && !is_power_of_2(pdata->ftrace_size))
497         pdata->ftrace_size = rounddown_pow_of_two(pdata->ftrace_size);
498     if (pdata->pmsg_size && !is_power_of_2(pdata->pmsg_size))
499         pdata->pmsg_size = rounddown_pow_of_two(pdata->pmsg_size);
500 
501     cxt->size = pdata->mem_size;			// 存crash的ram大小
502     cxt->phys_addr = pdata->mem_address;	// 存crash的内存物理地址
503     cxt->memtype = pdata->mem_type;
504     cxt->record_size = pdata->record_size;
505     cxt->console_size = pdata->console_size;
506     cxt->ftrace_size = pdata->ftrace_size;
507     cxt->pmsg_size = pdata->pmsg_size;
508     cxt->dump_oops = pdata->dump_oops;
509     cxt->ecc_info = pdata->ecc_info;
510 
511     paddr = cxt->phys_addr;
512 
    	/*
    		下面初始化四个内存保存到struct ramoops_context oops_cxt
    		 86 struct ramoops_context {
             87     struct persistent_ram_zone **dprzs; /* Oops dump zones */
             88     struct persistent_ram_zone *cprz;   /* Console zone */
             89     struct persistent_ram_zone **fprzs; /* Ftrace zones */
             90     struct persistent_ram_zone *mprz;   /* PMSG zone */
             91     phys_addr_t phys_addr;
             92     unsigned long size;
             93     unsigned int memtype;
             94     size_t record_size;
             95     size_t console_size;
             96     size_t ftrace_size;
             97     size_t pmsg_size;
             98     int dump_oops;
             99     u32 flags;
            100     struct persistent_ram_ecc_info ecc_info;
            101     unsigned int max_dump_cnt;
            102     unsigned int dump_write_cnt;
            103     /* _read_cnt need clear on ramoops_pstore_open */
            104     unsigned int dump_read_cnt;
            105     unsigned int console_read_cnt;
            106     unsigned int max_ftrace_cnt;
            107     unsigned int ftrace_read_cnt;
            108     unsigned int pmsg_read_cnt;
            109     struct pstore_info pstore;
            110 };
			
			 44 struct persistent_ram_zone {
             45     phys_addr_t paddr; 
             46     size_t size;
             47     void *vaddr;
             48     struct persistent_ram_buffer *buffer;
             49     size_t buffer_size;
             50     u32 flags;
             51     raw_spinlock_t buffer_lock;
             52 
             53     /* ECC correction */
             54     char *par_buffer;
             55     char *par_header;
             56     struct rs_control *rs_decoder;
             57     int corrected_bytes;
             58     int bad_blocks;
             59     struct persistent_ram_ecc_info ecc_info;
             60 
             61     char *old_log;
             62     size_t old_log_size;
             63 };
    	*/
        // dump_mem_sz的大小是设备树中定义的总的内存大小减去console+pmsg+ftrace
779     dump_mem_sz = cxt->size - cxt->console_size - cxt->ftrace_size
780             - cxt->pmsg_size;

		/*
			name:dump
			przs:&cxt->dprzs
			paddr:dts传过来的地址
			mem_sz:dump的大小,总大小-console-pmsg-ftrace
			record_size:cxt->record_size
			cnt:&cxt->max_dump_cnt
			sig:0
			flags:0
			528 static int ramoops_init_przs(const char *name,
            529                  struct device *dev, struct ramoops_context *cxt,
            530                  struct persistent_ram_zone ***przs,
            531                  phys_addr_t *paddr, size_t mem_sz,
            532                  ssize_t record_size,
            533                  unsigned int *cnt, u32 sig, u32 flags)
            534 {
            535     int err = -ENOMEM;
            536     int i;
            537     size_t zone_sz;
            538     struct persistent_ram_zone **prz_ar;
            539 
            540     /* Allocate nothing for 0 mem_sz or 0 record_size. */
            541     if (mem_sz == 0 || record_size == 0) {
            542         *cnt = 0;
            543         return 0;
            544     }
            545 
            546     /*
            547      * If we have a negative record size, calculate it based on
            548      * mem_sz / *cnt. If we have a positive record size, calculate
            549      * cnt from mem_sz / record_size.
            550      */
                	/*
                		pstore会将预留的内存按照record_size的大小进行chunk，该值会进行2的
                		幂次四舍五入
                	*/
            551     if (record_size < 0) {
            552         if (*cnt == 0)
            553             return 0;
            554         record_size = mem_sz / *cnt;
            555         if (record_size == 0) {
            556             dev_err(dev, "%s record size == 0 (%zu / %u)\n",
            557                 name, mem_sz, *cnt);
            558             goto fail;
            559         }
            560     } else {
                		// 我们设备树中定义的大于0（0x4000:16k）
                		// cnt记录了当前的dump zone按照record_size能分多少个chunk
            561         *cnt = mem_sz / record_size;
            562         if (*cnt == 0) {
            563             dev_err(dev, "%s record count == 0 (%zu / %zu)\n",
            564                 name, mem_sz, record_size);
            565             goto fail;
            566         }
            567     }
            568 
                	// *paddr + mem_sz代表当前dump zone的末尾物理地址
            569     if (*paddr + mem_sz - cxt->phys_addr > cxt->size) {
            570         dev_err(dev, "no room for %s mem region (0x%zx@0x%llx) in (0x%lx@0x%llx)\n",
            571             name,
            572             mem_sz, (unsigned long long)*paddr,
            573             cxt->size, (unsigned long long)cxt->phys_addr);
            574         goto fail;
            575     }
            576 
            577     zone_sz = mem_sz / *cnt;
            578     if (!zone_sz) {
            579         dev_err(dev, "%s zone size == 0\n", name);
            580         goto fail;
            581     }
            582 
                	/*
                		一个chunk用一个persistent_ram_zone结构表示
                		这里一个dump zone包含了cnt个chunk，就分配cnt个结构体
                	*/
            583     prz_ar = kcalloc(*cnt, sizeof(**przs), GFP_KERNEL);
            584     if (!prz_ar)
            585         goto fail;
            586 
                	/*
                		for循环为上面分配的每个persistent_ram_zone结构映射对应的虚拟地址，
                		此处dump zone有几个chunk就映射对应大小的物理地址到几个
                		persistent_ram_buffer
                         32 struct persistent_ram_buffer {
                         33     uint32_t    sig;
                         34     atomic_t    start;
                         35     atomic_t    size;
                         36     uint8_t     data[0];	// 真正存放数据的buffer
                         37 }; 
                		 44 struct persistent_ram_zone {
                         45     phys_addr_t paddr;	// 物理地址
                         46     size_t size;		// 大小
                         47     void *vaddr; 		// 虚拟地址
                         		// buffer就代表一个chunk的缓存，buffer指针指向映射之后的虚拟地址
                         48     struct persistent_ram_buffer *buffer;
                         49     size_t buffer_size; // 该chunk对应的persistent_ram_zone的大小要减去persistent_ram_buffer结构体的大小，就是每一个buffer除了本身的数据都会存一个persistent_ram_buffer结构体
                         50     u32 flags;
                         51     raw_spinlock_t buffer_lock;
                         52     
                         53     /* ECC correction */
                         54     char *par_buffer;
                         55     char *par_header;
                         56     struct rs_control *rs_decoder;
                         57     int corrected_bytes;
                         58     int bad_blocks;
                         59     struct persistent_ram_ecc_info ecc_info;
                         60 
                         61     char *old_log;
                         62     size_t old_log_size;
                         63 };
                	*/
            587     for (i = 0; i < *cnt; i++) {
                		/*
                			paddr：dump初始化时还是整个内存的起始物理地址
                			zone_sz：该dump zone每个chunk的大小，其实就是record_size
                			sig：0
                			ctx->memtype：unbuffered
                			
                			537 struct persistent_ram_zone *persistent_ram_new(phys_addr_t 
                							start, size_t size,
                            538             u32 sig, struct persistent_ram_ecc_info *ecc_info,
                            539             unsigned int memtype, u32 flags)
                            540 {
                            541     struct persistent_ram_zone *prz;
                            542     int ret = -ENOMEM;
                            543 
                            544     prz = kzalloc(sizeof(struct persistent_ram_zone), GFP_KERNEL);
                            545     if (!prz) {
                            546         pr_err("failed to allocate persistent ram zone\n");
                            547         goto err;
                            548     }
                            549 
                            550     /* Initialize general buffer state. */
                            551     raw_spin_lock_init(&prz->buffer_lock);
                            552     prz->flags = flags;
                            553 
                                	// 将start开始的物理地址，size大小按照page对齐映射到prz
                            554     ret = persistent_ram_buffer_map(start, size, prz, memtype);
                            555     if (ret)
                            556         goto err;
                            557     
                                	// 给persistent_ram_zone的ecc相关参数初始化
                            558     ret = persistent_ram_post_init(prz, sig, ecc_info);
                            559     if (ret)
                            560         goto err;
                            561     
                            562     return prz;
                            563 err:
                            564     persistent_ram_free(prz);
                            565  
                		*/
            588         prz_ar[i] = persistent_ram_new(*paddr, zone_sz, sig,
            589                           &cxt->ecc_info,
            590                           cxt->memtype, flags);
            591         if (IS_ERR(prz_ar[i])) {
            592             err = PTR_ERR(prz_ar[i]);
            593             dev_err(dev, "failed to request %s mem region (0x%zx@0x%llx): %d\n",
            594                 name, record_size,
            595                 (unsigned long long)*paddr, err);
            596 
            597             while (i > 0) {
            598                 i--;
            599                 persistent_ram_free(prz_ar[i]);
            600             }
            601             kfree(prz_ar);
            602             goto fail;
            603         }
                		// 映射一个chunk物理地址到persistent_ram_buffer，总的物理地址就偏移对应的
                		// 大小
            604         *paddr += zone_sz;
            605     }
            606 
            607     *przs = prz_ar;
            608     return 0;
            609 
            610 fail:
            611     *cnt = 0;
            612     return err;
            613 }
		*/
781     err = ramoops_init_przs("dump", dev, cxt, &cxt->dprzs, &paddr,
782                 dump_mem_sz, cxt->record_size,
783                 &cxt->max_dump_cnt, 0, 0);
784     pr_err("[mkk][%s]dump_vaddr[0]:%x\n", __FUNCTION__,cxt->dprzs[0]->vaddr);
785     if (err)
786         goto fail_out;
787 
788     err = ramoops_init_prz("console", dev, cxt, &cxt->cprz, &paddr,
789                    cxt->console_size, 0);
790     pr_err("[mkk][%s]console_vaddr:%x\n", __FUNCTION__, cxt->cprz->vaddr);
791     if (err)
792         goto fail_init_cprz;
793 
794     cxt->max_ftrace_cnt = (cxt->flags & RAMOOPS_FLAG_FTRACE_PER_CPU)
795                 ? nr_cpu_ids
796                 : 1;
797     err = ramoops_init_przs("ftrace", dev, cxt, &cxt->fprzs, &paddr,
798                 cxt->ftrace_size, -1,
799                 &cxt->max_ftrace_cnt, LINUX_VERSION_CODE,
800                 (cxt->flags & RAMOOPS_FLAG_FTRACE_PER_CPU)
801                     ? PRZ_FLAG_NO_LOCK : 0);
802     if (err)
803         goto fail_init_fprz;
804 
805     err = ramoops_init_prz("pmsg", dev, cxt, &cxt->mprz, &paddr,
806                 cxt->pmsg_size, 0);
807     if (err)
808         goto fail_init_mprz;
532 
533     cxt->pstore.data = cxt;	// 将ramoops_context结构本身放在cxt->pstore_info的私有域data中
809     /*
810      * Prepare frontend flags based on which areas are initialized.
811      * For ramoops_init_przs() cases, the "max count" variable tells
812      * if there are regions present. For ramoops_init_prz() cases,
813      * the single region size is how to check.
814      */
815     cxt->pstore.flags = 0;
816     if (cxt->max_dump_cnt)
817         cxt->pstore.flags |= PSTORE_FLAGS_DMESG;
818     if (cxt->console_size)
819         cxt->pstore.flags |= PSTORE_FLAGS_CONSOLE;
820     if (cxt->max_ftrace_cnt)
821         cxt->pstore.flags |= PSTORE_FLAGS_FTRACE;
822     if (cxt->pmsg_size)
823         cxt->pstore.flags |= PSTORE_FLAGS_PMSG;
824 
825     /*
826      * Since bufsize is only used for dmesg crash dumps, it
827      * must match the size of the dprz record (after PRZ header
828      * and ECC bytes have been accounted for).
829      */
830     if (cxt->pstore.flags & PSTORE_FLAGS_DMESG) {
831         cxt->pstore.bufsize = cxt->dprzs[0]->buffer_size;
832         cxt->pstore.buf = kzalloc(cxt->pstore.bufsize, GFP_KERNEL);
833         if (!cxt->pstore.buf) {
834             pr_err("cannot allocate pstore crash dump buffer\n");
835             err = -ENOMEM;
836             goto fail_clear;
837         }
838     }
550 
    	// pstore_register中会将kmsg_dumpe注册到全局的dump_list，ksmg_dump会遍历dump_list调用 dump函数 
    	/*
    		494 static struct ramoops_context oops_cxt = {
            495     .pstore = {
            496         .owner  = THIS_MODULE,
            497         .name   = "ramoops",
            498         .open   = ramoops_pstore_open,
            499         .read   = ramoops_pstore_read,
            500         .write  = ramoops_pstore_write,
            501         .write_user = ramoops_pstore_write_user,
            502         .erase  = ramoops_pstore_erase,
            503     },
            504 };
    	*/
551     err = pstore_register(&cxt->pstore);
552     if (err) {
553         pr_err("registering with pstore failed\n");
554         goto fail_buf;
555     }
556 
557     /*
558      * Update the module parameter variables as well so they are visible
559      * through /sys/module/ramoops/parameters/
560      */
561     mem_size = pdata->mem_size;
562     mem_address = pdata->mem_address;
563     record_size = pdata->record_size;
564     dump_oops = pdata->dump_oops;
565     ramoops_console_size = pdata->console_size;
566     ramoops_pmsg_size = pdata->pmsg_size;
567     ramoops_ftrace_size = pdata->ftrace_size;
568 
569     pr_info("attached 0x%lx@0x%llx, ecc: %d/%d\n",
570         cxt->size, (unsigned long long)cxt->phys_addr,
571         cxt->ecc_info.ecc_size, cxt->ecc_info.block_size);
572 
573     return 0;
588 }

fs/pstore/platform.c
441 int pstore_register(struct pstore_info *psi)
442 {
443     struct module *owner = psi->owner;
444 
445     if (backend && strcmp(backend, psi->name))
446         return -EPERM;
447 
448     spin_lock(&pstore_lock);
449     if (psinfo) {
450         spin_unlock(&pstore_lock);
451         return -EBUSY;
452     }
453 
    	// 这里将pstore_write_compat赋值给pstore_info->write
454     if (!psi->write)
455         psi->write = pstore_write_compat;
456     psinfo = psi;	// 初始化全局的psinfo，这个是probe中的ctx的pstore成员
457     mutex_init(&psinfo->read_mutex);
458     spin_unlock(&pstore_lock);
459 
460     if (owner && !try_module_get(owner)) {
461         psinfo = NULL;
462         return -EINVAL;
463     }
464 
465     allocate_buf_for_compression();
466 
467     if (pstore_is_mounted()) // 判断pstore文件系统是否挂载,在inode.c中会挂载pstore文件系统
468         pstore_get_records(0); // 读取预留ram的内容到/sys/fs/pstore/xxx
    	/*
    		------------> inode.c
    		414 /*  
            415  * Read all the records from the persistent store. Create
            416  * files in our filesystem.  Don't warn about -EEXIST errors
            417  * when we are re-scanning the backing store looking to add new
            418  * error records.
            419  */
            420 void pstore_get_records(int quiet)
            421 {
            422     struct pstore_info *psi = psinfo;
            423     struct dentry *root;
            424 
            425     if (!psi || !pstore_sb)
            426         return;
            427 
            428     root = pstore_sb->s_root;
            429 
            430     inode_lock(d_inode(root));
            431     pstore_get_backend_records(psi, root, quiet);
            432     inode_unlock(d_inode(root));
            433 }
    
    		---------> platform.c
    		811 /*
            812  * Read all the records from one persistent store backend. Create
            813  * files in our filesystem.  Don't warn about -EEXIST errors
            814  * when we are re-scanning the backing store looking to add new
            815  * error records.
            816  */
            817 void pstore_get_backend_records(struct pstore_info *psi,
            818                 struct dentry *root, int quiet)
            819 {   
            820     int failed = 0;
            821     unsigned int stop_loop = 65536;
            822     
            823     if (!psi || !root)
            824         return;
            825 
            826     mutex_lock(&psi->read_mutex);
            827     if (psi->open && psi->open(psi))
            828         goto out;
            829 
            830     /*
            831      * Backend callback read() allocates record.buf. decompress_record()
            832      * may reallocate record.buf. On success, pstore_mkfile() will keep
            833      * the record.buf, so free it only on failure.
            834      */ 
            835     for (; stop_loop; stop_loop--) {
            836         struct pstore_record *record;
            837         int rc;
            838 
            839         record = kzalloc(sizeof(*record), GFP_KERNEL);
            840         if (!record) {
            841             pr_err("out of memory creating record\n");
            842             break;
            843         }
            844         pstore_record_init(record, psi);
            845         
                		// 调用ram.c中pstore_info结构中的read函数从ram中读内容
            846         record->size = psi->read(record);
            847 
            848         /* No more records left in backend? */
            849         if (record->size <= 0) {
            850             kfree(record);
            851             break;
            852         }
            853 		
                		// 解压从ram读出来的内容
            854         decompress_record(record);
                		// 负责在/sys/fs/pstore节点（pstore文件系统root）创建pstore文件
            855         rc = pstore_mkfile(root, record);
            856         if (rc) {
            857             /* pstore_mkfile() did not take record, so free it. */
            858             kfree(record->buf);
            859             kfree(record);
            860             if (rc != -EEXIST || !quiet)
            861                 failed++;
            862         }
            863     }
            864     if (psi->close)
            865         psi->close(psi);
            866 out:
            867     mutex_unlock(&psi->read_mutex);
            868 
            869     if (failed)
            870         pr_warn("failed to create %d record(s) from '%s'\n",
            871             failed, psi->name);
            872     if (!stop_loop)
            873         pr_err("looping? Too many records seen from '%s'\n",
            874             psi->name);
            875 }
    	*/
469
    	
711     if (psi->flags & PSTORE_FLAGS_DMESG)
712         pstore_register_kmsg();	// 内部注册kmsg_dumper，panic、reboot时kmsg_dump调用
		
    	/*
    		https://blog.csdn.net/tiantao2012/article/details/52352426
    		注册console，prink中最后会调用call_console_drivers 来将log通过uart 输出
    		
    		console的pstore_console_write函数会将printk的内容输出到ram
    	*/
713     if (psi->flags & PSTORE_FLAGS_CONSOLE)
714         pstore_register_console()
715     if (psi->flags & PSTORE_FLAGS_FTRACE)
716         pstore_register_ftrace();
    	/*
    		pstore_register_pmsg会在dev下生成pmsg文件
    		字符设备的file_operations的write函数write_pmsg会将用户的信息写入pstore保留的内存
    	*/
717     if (psi->flags & PSTORE_FLAGS_PMSG)
718         pstore_register_pmsg();
719 
720     /* Start watching for new records, if desired. */
721     if (pstore_update_ms >= 0) {
722         pstore_timer.expires = jiffies +
723             msecs_to_jiffies(pstore_update_ms);
724         add_timer(&pstore_timer);
725     }
726 
727     /*
728      * Update the module parameter backend, so it is visible
729      * through /sys/module/pstore/parameters/backend
730      */
731     backend = psi->name;
732 
733     pr_info("Registered %s as persistent store backend\n", psi->name);
734 
735     module_put(owner);
736 
737     return 0;
738 }

fs/pstore/platform.c
360 static struct kmsg_dumper pstore_dumper = {
361     .dump = pstore_dump,
362 };
367 static void pstore_register_kmsg(void)
368 {
369     kmsg_dump_register(&pstore_dumper);
370 }

/*
	pstore_dump就是panic时kmsg_dump遍历dump_list调用pstore_dumper的dump方法
*/
279 static void pstore_dump(struct kmsg_dumper *dumper,
280             enum kmsg_dump_reason reason)
281 {
282     unsigned long   total = 0;
283     const char  *why;
284     u64     id;
285     unsigned int    part = 1;
286     unsigned long   flags = 0;
287     int     is_locked = 0;
288     int     ret;
289 
290     why = get_reason_str(reason);	// 获取dump原因（restart|panic...）
291 
292     if (pstore_cannot_block_path(reason)) {
293         is_locked = spin_trylock_irqsave(&psinfo->buf_lock, flags);
294         if (!is_locked) {
295             pr_err("pstore dump routine blocked in %s path, may corrupt error record\n"
296                        , in_nmi() ? "NMI" : why);
297         }
298     } else
299         spin_lock_irqsave(&psinfo->buf_lock, flags);
300     oopscount++;
301     while (total < kmsg_bytes) {
302         char *dst;
303         unsigned long size;
304         int hsize;
305         int zipped_len = -1;
306         size_t len;
307         bool compressed;
308         size_t total_len;
309 
310         if (big_oops_buf && is_locked) {
311             dst = big_oops_buf;
312             hsize = sprintf(dst, "%s#%d Part%u\n", why,
313                             oopscount, part);
314             size = big_oops_buf_sz - hsize;
315 
316             if (!kmsg_dump_get_buffer(dumper, true, dst + hsize,
317                                 size, &len))
318                 break;
319 
320             zipped_len = pstore_compress(dst, psinfo->buf,
321                         hsize + len, psinfo->bufsize);
322 
323             if (zipped_len > 0) {
324                 compressed = true;
325                 total_len = zipped_len;
326             } else {
327                 compressed = false;
328                 total_len = copy_kmsg_to_buffer(hsize, len);
329             }
330         } else {
331             dst = psinfo->buf;
332             hsize = sprintf(dst, "%s#%d Part%u\n", why, oopscount,
333                                     part);
334             size = psinfo->bufsize - hsize;
335             dst += hsize;
336 
337             if (!kmsg_dump_get_buffer(dumper, true, dst,
338                                 size, &len))
339                 break;
340 
341             compressed = false;
342             total_len = hsize + len;
343         }
344 
    		// psinfo是struct pstore_info结构，调用它的write方法保存内存信息
345         ret = psinfo->write(PSTORE_TYPE_DMESG, reason, &id, part,
346                     oopscount, compressed, total_len, psinfo);
347         if (ret == 0 && reason == KMSG_DUMP_OOPS && pstore_is_mounted())
348             pstore_new_entry = 1;
349 
350         total += total_len;
351         part++;
352     }
353     if (pstore_cannot_block_path(reason)) {
354         if (is_locked)
355             spin_unlock_irqrestore(&psinfo->buf_lock, flags);
356     } else
357         spin_unlock_irqrestore(&psinfo->buf_lock, flags);
358 }
359 
360 static struct kmsg_dumper pstore_dumper = {
361     .dump = pstore_dump,
362 };
```

<font color="red">从上面的分析可以得出：</font>

<font color="red">	1、panic或者reboot的dump信息会在crash时写入pstore</font>

<font color="red">	2、console的信息会在printk的时候写入pstore</font>

<font color="red">	3、pmsg的信息会在用户触发写/dev/pmsg0设备文件的时候写入pstore</font>

​	<font color="pink">在设备树中配置的pstore对应的各个区（有的可能没有配置）都是大小不足时循环写，每个区的大小是连续的，每个区都有一个id标识该区</font>

​	<font color="green">所以如果在掉电时为了获取当时的信息有两种方法，修改pstore写内存为写flash或者修改xbl在异常重启时将内存中数据写入flash，冷启动时将flash中的数据写入/sys/fs/pstore/xxx中</font>

