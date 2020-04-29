参考：
> https://segmentfault.com/a/1190000008268803


### １、文件系统配置  
```
1、/proc/sys/vm/panic_on_oom
		2：申请不到内存一定会出发panic
		1：有可能触发panic
		0：继续检查/proc/sys/vm/oom_kill_allocating_task文件
		
2、/proc/sys/vm/oom_kill_allocating_task
		1：kill掉当前申请内存的进程 
		0：内核将检查每个进程的分数，分数最高（占用内存最大的）的进程将被kill掉
		
3、当被kill掉之后如果/proc/sys/vm/oom_dump_tasks为1且系统的rlimit中设置了core文件大小，将会由/proc/sys/kernel/core_pattern里面指定的程序生成core dump文件，这个文件里将包含pid, uid, tgid, vm size, rss, nr_ptes, nr_pmds, swapents, oom_score_adj、score, name等内容，拿到这个core文件之后，可以做一些分析，看为什么这个进程被选中kill掉。

4、/proc/sys/kernel/panic
		发生panic之后系统重启的时间，单位秒
		
5、/proc/<pid>/
		oom_adj：oom_score中的值会根据这个值来调整（-17->15）
		oom_score：oom发生时进程的分数
```

