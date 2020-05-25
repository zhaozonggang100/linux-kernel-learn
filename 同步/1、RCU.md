### 1、RCU概述  

```
reference：
https://www.cnblogs.com/qcloud1001/p/7755331.html
http://abcdxyzk.github.io/blog/2015/07/31/kernel-sched-rcu/
https://cloud.tencent.com/developer/article/1518203
pdf：笨叔叔的奔跑吧Linux内核
```

RCU分类  
```
1、tiny rcu  
2、tree rcu
3、可睡眠rcu
4、可抢占rcu
```

### 2、RCU引发问题

> 1、stall on cpu then oom and call trace <__switch_to> 

```
reserence：
https://www.kernel.org/doc/Documentation/RCU/stallwarn.txt
http://www.itkeyword.com/doc/9445873753770208x824/rcu-preempt-self-detected-stall-on-cpu-0
https://access.redhat.com/solutions/2260151
https://stackoverflow.com/questions/35401317/rcu-preempt-self-detected-stall-on-cpu-0
https://unix.stackexchange.com/questions/252045/rcu-preempt-detected-stalls-on-cpus-tasks-message-appears-to-continue
https://www.cnblogs.com/aspirs/p/12633138.html
https://forum.ubuntu.org.cn/viewtopic.php?t=489786		<__switch_to> 
https://www.cnblogs.com/coder51up/p/6940030.html?utm_source=itdadao&utm_medium=referral 
		上面是ARM64没有backtrace只有exception时如何分析问题
http://ftp.uni-kl.de/pub/linux/kernel/v4.x/ChangeLog-4.14.45

https://bbs.csdn.net/topics/390981832?page=1		【cpu cache】
	cpu执行的指令和数据是从L1高速缓存的指令缓存和数据缓存中获取，一旦cpu要执行的指令或数据无法从高速缓存中获取，就会产生cpu stall。你这种情况是程序要求的cpu性能要高于你现在使用的cpu，你得考虑升级设备才能运行此庞大程序。
	再补充一点，当cpu无法从高速缓存中获取指令或数据，那么只有从内存中获取，而这种会浪费很长时间的，所以你的程序会hang住，考虑玩个小程序吧，或者非要玩这个大程序，换个牛逼手机吧。
	
https://app-community.nxp.com/thread/524191			【usb】
	usb xhci event报错导致rcu，去掉4.14.98的commit【077506972ba23772b752e08b1ab7052cf5f04511】并且添加usb的patch  fixed
	
https://blog.csdn.net/dog250/article/details/5303584
	NOHZ配置的kernel中，跳过关闭cpu心跳中task（假设它不可能在临界区）
	
疑点：
1、在kworker/1:0执行work时有read_rcu_lock临界区被抢占了
2、kworker/1:0本身执行异常阻塞主cpu了，使得cpu的时钟中断无法通知rcu_node清楚cpumask
3、cpu cache异常
```


### 3、各种rcu  
1、抢占式RCU和非抢占式RCU  

2、SRCU  

```
可睡眠RCU
```

3、tiny rcu  

4、tree rcu  

```
数据结构：
	struct rcu_node;
	struct rcu_data;
	struct rcu_state;			// PER-CPU，三种：
								//  rcu_sched_state:普通进程上下文切换	
								// 	rcu_preempt_state：开启抢占默认
								//	rcu_bh_state：中断下半部
								// 系统会初始化三个state用于不同场景
								// 每个state都包含一套rcu层次结构
config：
	CONFIG_RCU_FANOUT=64		// 每层支持的最多叶子数量
	CONFIG_RCU_FANOUT_LEAF=16	// 一个子叶子的CPU数量
	CONFIG_RCU_PREEMPT			// 开启rcu抢占，此时rcu_state对应的是
								// rcu_preempt_state，在rcu_read_lock
								// 临界区允许抢占
kernel/rcu/rcu.h
	MAX_RCU_LVLS				// rcu tree最多支持的层数
	
三个rcu_state对应到三个内核线程
```

### 4、RCU中的一些概念  

1、grace period  
```
RCU中一个非常重要的概念：宽限期，是指在写临界区中把拷贝的临时变量修改完之后，writer会publish，等待所有在写临界区之前进入的读临界区结束，就是给这些读临界区一个宽限期，当宽限期到了（默认21s,/sys/module/rcupdate/parameters/rcu_cpu_stall）之后还没有退出读临界区，硬件定时器触发软件定时器再触发内核线程rcu_preempt就会发出cpu stalls的警告，如果读临界区一直不退出执行rcu_read_unlock会不停的警告，每次时间都在递增
```

2、quiescent state  
```
当处在读临界区的进程退出临界区，他所在的cpu会发生一次上下文切换，qs就代表当前进程所在的cpu发生了一次上下文切换，当读者在临界区的时候认为当前cpu时活跃的，如果在gp之后的时钟中断中检测到该cpu处于用户模式或者idle模式，就认为发生了一次上下文切换
```

3、tree rcu中的状态机

### 5、API  
1、static __always_inline void rcu_read_lock(void)  
```
功能：进入读临界区
```

2、static inline void rcu_read_unlock(void)  
```
功能：退出读临界区
```

3、#define rcu_dereference(p) rcu_dereference_check(p, 0)  
```
功能：在读临界区，传入指针p，返回p的临时拷贝  
```

4、void synchronize_rcu(void)  
```
功能：在写临界区拷贝保护变量并修改拷贝之后，发出同步通知，硬件定时器会触发软件定时器进而触发内核线程rcu_preempt去检测在写临界区之前进入的读临界区是否全部结束，这里会阻塞等待
```

5、void call_rcu(struct rcu_head *head, rcu_callback_t func)  
```
功能：和synchronize_rcu类似，只是这里是注册回调函数，然后退出写临界区
```

6、#define rcu_assign_pointer(p, v)  
```
功能：在读临界区的时候，不能直接去读共享数据，需要调用该函数获取共享数据的指针拷贝，及p、v指向同一个区域
```

7、static inline void list_add_rcu(struct list_head *new, struct list_head *head)  【增】
```
功能：rcu在内核中一般是用来保护链表中的数据的，所以专门定义了操作链表的接口
```

8、static inline void list_del_rcu(struct list_head *entry)  【删】


9、static inline void list_replace_rcu(struct list_head *old, struct list_head *new)【改】   

10、#define list_entry_rcu(ptr, type, member)  container_of(READ_ONCE(ptr), type, member)【遍历】  

### 6、rcu的调试  
1、打开rcu的trace可以看到rcu的状态机变化  
2、/sys/modules/rcupdate/parameters/rcu_cpu_stall_suppress，rcu stall detector  
3、/sys/modules/rcupdate/parameters/rcu_cpu_stall_timeout，rcu stall message ouput 时间  ，默认是21s 



### 7、RCU的内核配置

CONFIG_PREEMPT = n和CONFIG_SMP = y意味着选择了CONFIG_TREE_RCU配置,因此选择非抢占式树型RCU的实现，比较适合SMP服务器级别的构建。 

CONFIG_PREEMPT_VOLUNTARY：自愿被抢占 



### 8、rcu初始化

start_kernel-->rcu_init



### 9、RCU调试

```
CONFIG_RCU_CPU_STALL_TIMEOUT
```

/sys/module/rcupdate/parameters/rcu_cpu_stall_suppress：这个检测会过度的延迟GP，系统上是0，没有使能

/sys/module/rcupdate/parameters/rcu_cpu_stall_timeout

```
RCU_STALL_DELAY_DELTA
```

由于lockdep带来的负载给RCU STALL检测延时5s

```
RCU_STALL_RAT_DELAY
```

本地cpu没有触发rcu超时检测，其他cpu会抱怨