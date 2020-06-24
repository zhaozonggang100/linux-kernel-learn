reference：

http://gityuan.com/2016/09/17/android-lowmemorykiller/

### 1、概述

Android通过内核提供的drivers/staging/android/lowmemorykiller.c驱动接口来实现内存不足时kill Android创建的进程。

Android中zygote孵化出来的进程都由ActivityManagerService管理，ActivityManagerService会调用lmkd将进程的oom_score_adj更新到proc中，接着进入内核执行进程扫描杀进程。

### 2、杀进程回收内存的最终函数

https://www.jianshu.com/p/4dbe9bbe0449

函数名：lowmen_scan

定义：drivers/staging/android/lowmemorykiller.c

```
183 static struct shrinker lowmem_shrinker = {
184     .scan_objects = lowmem_scan,
185     .count_objects = lowmem_count,
186     .seeks = DEFAULT_SEEKS * 16
187 };
```

### 3、杀进程时机

lmk的ActivityManagerService服务会周期性的扫描系统的内存，当内存低于Android启动时设置的阈值，就会触发杀进程过程。

### 4、lmkd进程

lmkd进程是由init进程根据init.rc中的定义启动的，它会创建/dev/socket/lmkd的socket  framework层交互

定义：system/core/lmkd/lmkd.c

### 5、设备节点

- sys/module/lowmemorykiller/parameters/minfree

eg：18432,23040,27648,32256,55296,80640

当要杀进程时判断当前的内存值在上面哪个范围内（单位page），然后在下面的adj对应区间找adj值最大的进程进行kill



- sys/module/lowmemorykiller/parameters/adj（oom_score_adj）

eg：0,100,200,250,900,950