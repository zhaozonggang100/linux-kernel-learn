### 1、preempt_count

```
struct task_struct {
#ifdef CONFIG_PREEMPT_RCU
		int rcu_read_lock_nesting;				// tree rcu 在执行rcu_read_lock时会对该值++
#endif
		struct thread_info;
}

struct thread_info {
		union {
				u64     preempt_count;  /* 0 => preemptible, <0 => bug */
         		struct {
 #ifdef CONFIG_CPU_BIG_ENDIAN
             			u32 need_resched;
             			u32 count;
 #else
             			u32 count;
             			u32 need_resched;
 #endif
 				} preempt;
		}
}
```
进程的preempt_count代表进程是否可被抢占  
preempt_count位图  

<table>
    <tr>
        <td>63bit</td>
        <td>保留</td>
        <td>20bit</td>
        <td>16~19bit</td>
        <td>8~15bit</td>
        <td>0~7bit</td>
    </tr>
    <tr>
    	<td>PREEMPT_NEED_RESCHED</td>
         <td></td>
        <td>NMI_MASK</td>
        <td>HARDIRQ_MASK</td>
        <td>SOFTIRQ_MASK</td>
        <td>PREEMPT_MASK</td>
    </tr>

bits 0-7 are the preemption count (max preemption depth: 256)

bits 8-15 are the softirq count (max # of softirqs: 256)，单cpu上不允许softirq嵌套，所以也就只用到bit15

bit 16~19 hardirq count，由于Linux不支持hardirq嵌套，所以只有bit16起作用



### 2、preempt相关函数  

1、preempt_disable()

```
关闭当前进程抢占一次，最多嵌套关闭256次（8bits），如果当前进程要被抢占必须对应的执行preempt_enable相应次数：
		u32 pc = READ_ONCE(current_thread_info()->preempt.count);
		pc += val;
 		WRITE_ONCE(current_thread_info()->preempt.count, pc);
```

2、preempt_schedule

```
https://www.sohu.com/a/229979296_467784
判断当前进程是否可以主动出让cpu给其他线程
```

3、void __wake_up_sync_key(wait_queue_head_t *q, unsigned int mode, int nr, void *key);

```
https://zhuanlan.zhihu.com/p/47312156
```

