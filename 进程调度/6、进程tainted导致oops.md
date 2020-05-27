参考：https://blog.csdn.net/iteye_21199/article/details/82126323

### 1、概述

进程的异常行为会导致内核被tainted（污染），从而导致OOPS，有时候会宕机

### 2、实例分析

```
CPU: 3 PID: 510 Comm: vold Tainted: P        W  O    4.4.138-user+ #3
```

进程vold导致了内核被污染，被污染时内核会打印出污染进程信息，以及call  trace

```
/**
 *  print_tainted - return a string to represent the kernel taint state.
 *
 *  'P' - Proprietary module has been loaded.
 *  'F' - Module has been forcibly loaded.
 *  'S' - SMP with CPUs not designed for SMP.
 *  'R' - User forced a module unload.
 *  'M' - System experienced a machine check exception.
 *  'B' - System has hit bad_page.
 *  'U' - Userspace-defined naughtiness.
 *  'D' - Kernel has oopsed before
 *  'A' - ACPI table overridden.
 *  'W' - Taint on warning.
 *  'C' - modules from drivers/staging are loaded.
 *  'I' - Working around severe firmware bug.
 *  'O' - Out-of-tree module has been loaded.
 *  'E' - Unsigned module has been loaded.
 *  'L' - A soft lockup has previously occurred.
 *  'K' - Kernel has been live patched.
 *
 *  The string is overwritten by the next call to print_tainted().
 */
```

上面的字母代表污染的原因