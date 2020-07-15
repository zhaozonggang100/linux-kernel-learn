### 1、系统调用表

```c
include/uapi/asm-generic/unistd.h
    
421 /* kernel/sys.c */
426 #define __NR_reboot 142
427 __SYSCALL(__NR_reboot, sys_reboot)
```



