reference：

http://gityuan.com/2016/09/17/android-lowmemorykiller/

https://www.jianshu.com/p/4dbe9bbe0449

### 1、概述

Android通过内核提供的drivers/staging/android/lowmemorykiller.c驱动接口来实现内存不足时kill Android创建的进程。

Android中zygote孵化出来的进程都由ActivityManagerService管理，ActivityManagerService会调用lmkd将进程的oom_score_adj更新到proc中，接着进入内核执行进程扫描杀进程。

### 2、lowmemorykiller框架

lowmemorykiller主要由三部分组成

1、AMS部分的ProcessList（framework）

2、Native进程lmkd（native）

3、内核中的LowMemoryKiller部分（kernel）



framework层的ProcessList和native层的lmkd通过socket（/dev/socket/lmkd）通信

lmkd和内核中的LowMemoryKiller通过writeFileString向文件节点写内容方法进行通信



lmkd杀进程方式：

1、将阀值写入文件节点发送给内核的LowMemoryKiller，由内核进行杀进程处理（4.12中已经移除）

2、通过cgroup监控内存使用情况，自行计算杀掉进程。



> 为了防止剩余内存过低，Android在内核空间有lowmemorykiller(简称LMK)，LMK是通过注册shrinker来触发低内存回收的，这个机制并不太优雅，可能会拖慢Shrinkers内存扫描速度，已从内核4.12中移除，后续会采用用户空间的LMKD + memory cgroups机制。



### 3、LMKD如何处理来自framework层的连接以及cmd

1、创建socket

2、epoll监听socket，排除中断和信号的干扰

3、调用ctrl_data_handler



> framework发送给lmkd的cmd：

1、cmd_target 调整最小内存阀值和adj值

会将两个值写入到文件系统节点：

​	/sys/module/lowmemorykiller/parameters/minfree		18432,23040,27648,32256,55296,80640

​	/sys/module/lowmemorykiller/parameters/adj				  0,100,200,250,900,950

2、cmd_procprio 调整进程的adj值

调整进程的被杀优先级（/proc/pid/oom_score_adj）

3、cmd_procremove 移除对应的进程

由内核处理



```
lmdk源码：system/core/lmkd/lmkd.c
```



### 4、kernel  lowmemorykiller处理

```
源码：drivers/staging/android/lowmemorykiller.c
```

