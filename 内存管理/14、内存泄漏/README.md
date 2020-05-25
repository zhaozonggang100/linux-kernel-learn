### 1、用户态内存泄漏  
使用工具：memleak、valgrind  

> 1、ubuntu memleak使用  

```
1、安装  
	sudo apt install bpfcc-tools或者直接github下载源码
	memleak属于bcc-tools软件包的一个工具
```

### 2、内核态内存泄露  
kmemleak：https://blog.csdn.net/zhuyong006/article/details/83089407

1、开启内核配置

CONFIG_HAVE_DEBUG_KMEMLEAK=y
CONFIG_DEBUG_KMEMLEAK=y

CONFIG_DEBUG_KMEMLEAK_DEFAULT_OFF=y				// 需要手动开启kmemleak检测  echo scan > /sys/kernel/debug/kmemleak

CONFIG_DEBUG_KMEMLEAK_EARLY_LOG_SIZE=400     // kmemleak文件大小

2、修改cmdline

在uboot的bootarg中加入"kmemleak=on"

3、查看kmemleak检测到的内存泄露

cat /sys/kernel/debug/kmemleak

