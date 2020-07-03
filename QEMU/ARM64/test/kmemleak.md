### 1、概述

使用kmemleak测试内存泄露，kmemleak是通过debugfs接口通过sysfs文件系统提供给用户使用

### 2、内核配置(.config)

CONFIG_DEBUG_KMEMLEAK=y

CONFIG_DEBUG_KMEMLEAK_EARLY_LOG_SIZE=20000

CONFIG_DEBUG_KMEMLEAK_DEFAULT_OFF=y

### 3、bootloader给kernel的传参

在RunBiscuitOS.sh的append参数中添加`kmemleak=on`

这样会在系统启动时，把kmemleak添加到debugfs

### 4、测试内存泄露

1、自己编写一个驱动模块，下面是我写的一个简单测试例子

```c
  1 #include <linux/module.h>
  2 #include <linux/slab.h>
  3 
  4 static int __init akon_kmemleak_init(void)
  5 {
  6     char *amem = NULL;
  7     printk(KERN_INFO "Entry kmemleak test\n");
  8     amem = kmalloc(32,GFP_KERNEL);
  9     return 0;
 10 }
 11 
 12 static void __exit akon_kmemleak_eixt(void)
 13 {
 14     printk("test kmemleak exit");
 15 }
 16 
 17 
 18 MODULE_LICENSE("GPL");
 19 MODULE_AUTHOR("Akon");
 20 MODULE_DESCRIPTION("Kmemleak test");
 21 module_init(akon_kmemleak_init);
 22 module_exit(akon_kmemleak_eixt);
```

2、手动挂载`BiscuitOS/output/linux-4.4.174-aarch`目录下的BiscuitOS.img，将自己编写的ko文件cp到文件系统中，然后卸载掉（或者按照张教主的方法重新生成BiscuitOS.img）

3、./RunBiscuitOS.sh

4、查看/sys/kernel/debug/kmemleak文件是否存在，存在说明已经正常使能了kmemleak

5、kmemleak默认10分钟scan一次，这里我们选择手动触发

```
echo scan > /sys/kernel/debug/kmemleak
```

6、等待console有kmemleak信息输出之后，执行下面命令可以看到导致泄露的根源

```
cat /sys/kernel/debug/kmemleak
```

