### 1、概述

migrations是用来cpu之间做负载均衡的



[https://blog.csdn.net/zhanggang807/article/details/72828358?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522159650348819195188363079%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=159650348819195188363079&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v3~pc_rank_v3-1-72828358.pc_ecpm_v3_pc_rank_v3&utm_term=Linux+migration&spm=1018.2118.3001.4187](https://blog.csdn.net/zhanggang807/article/details/72828358?ops_request_misc=%7B%22request%5Fid%22%3A%22159650348819195188363079%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=159650348819195188363079&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v3~pc_rank_v3-1-72828358.pc_ecpm_v3_pc_rank_v3&utm_term=Linux+migration&spm=1018.2118.3001.4187)

### 2、初始化

 rest_init-->kernel_init-->kernel_init_freeable-->do_pre_smp_initcalls，接着初始化module

```c
// file：kernel/sched/core.c

6421 static int __init migration_init(void)
6422 {
6423     void *cpu = (void *)(long)smp_processor_id();
6424     int err;
6425 
    	 /*
    	 	创建migration/x内核线程
    	 */
6426     /* Initialize migration for the boot CPU */
6427     err = migration_call(&migration_notifier, CPU_UP_PREPARE, cpu);
6428     BUG_ON(err == NOTIFY_BAD);
6429     migration_call(&migration_notifier, CPU_ONLINE, cpu);
6430     register_cpu_notifier(&migration_notifier);
6431     
6432     /* Register cpu active notifiers */
6433     cpu_notifier(sched_cpu_active, CPU_PRI_SCHED_ACTIVE);
6434     cpu_notifier(sched_cpu_inactive, CPU_PRI_SCHED_INACTIVE);
6435 
6436     return 0;
6437 }   
6438 early_initcall(migration_init);
```

