### 1、用户态内存泄漏  
使用工具：memleak、valgrind  

> 1、ubuntu memleak使用  

```
1、安装  
	sudo apt install bpfcc-tools或者直接github下载源码
	memleak属于bcc-tools软件包的一个工具
```

### 2、内核态内存泄露  
kmemleak：

​		https://blog.csdn.net/zhuyong006/article/details/83089407

​		 https://www.jianshu.com/p/b0bfc2b97ae3 

​		  [https://www.jeanleo.com/2018/09/09/%E3%80%90linux%E5%86%85%E5%AD%98%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E3%80%91%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E6%A3%80%E6%B5%8Bkmemleak%E5%88%86%E6%9E%90/](https://www.jeanleo.com/2018/09/09/[linux内存源码分析]内存泄漏检测kmemleak分析/) 

1、开启内核配置

CONFIG_HAVE_DEBUG_KMEMLEAK=y
CONFIG_DEBUG_KMEMLEAK=y

CONFIG_DEBUG_KMEMLEAK_DEFAULT_OFF=n				// 需要手动开启kmemleak检测  echo scan > /sys/kernel/debug/kmemleak

CONFIG_DEBUG_KMEMLEAK_EARLY_LOG_SIZE=400     // kmemleak文件大小

2、修改cmdline

在uboot的bootarg中加入"kmemleak=on"

3、查看kmemleak检测到的内存泄露

cat /sys/kernel/debug/kmemleak



### 3、kmemleak原理

 http://blog.chinaunix.net/uid-26859697-id-5758036.html 

kmemleak的原理其实就是通过kmalloc、vmalloc、kmem_cache_alloc等内存的分配，跟踪其指针，连同其他的分配大小和堆栈跟踪信息，存储在PRIO搜索树。如果存在相应的释放函数调用跟踪和指针，就会从kmemleak数据结构中移除。