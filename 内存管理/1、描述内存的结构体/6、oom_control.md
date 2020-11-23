### 1、结构体定义

```c
 15 /*  
 16  * Details of the page allocation that triggered the oom killer that are used to
 17  * determine what should be killed.
 18  */     
 19 struct oom_control {
 20     /* Used to determine cpuset */
 21     struct zonelist *zonelist;
 22  
 23     /* Used to determine mempolicy */
 24     nodemask_t *nodemask;
 25     
 26     /* Used to determine cpuset and node locality requirement */
 27     const gfp_t gfp_mask; // 发生异常时申请页面order大小。
 28                 
 29     /*
 30      * order == -1 means the oom kill is required by sysrq, otherwise only
 31      * for display purposes.
 32      */
 33     const int order; // 发生异常时申请页面order大小
 34 };
```

