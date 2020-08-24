### 1、reboot调用流程

- 1、libc库通过系统调用产生软中断进入kernel，kernel中arm64的实现查看系统调用表

```
具体怎么从用户态进入内核态查找系统调用表的过程参考write系统调用

系统调用号在系统调用表中定义：include/uapi/asm-generic/unistd.h
423 #define __NR_reboot 142
424 __SYSCALL(__NR_reboot, sys_reboot)
reboot的系统调用号是142
```

- 2、实现reboot系统调用在kernel/reboot.c中

```c
272 /*
273  * Reboot system call: for obvious reasons only root may call it,
274  * and even root needs to set up some magic numbers in the registers
275  * so that some mistake won't make this reboot the whole machine.
276  * You can also set the meaning of the ctrl-alt-del-key here.
277  *
278  * reboot doesn't sync: do that yourself before calling this.
279  */
280 SYSCALL_DEFINE4(reboot, int, magic1, int, magic2, unsigned int, cmd,
281         void __user *, arg)
282 {
283     struct pid_namespace *pid_ns = task_active_pid_ns(current);
284     char buffer[256];
285     int ret = 0;
286
287     /* We only trust the superuser with rebooting the system. */
288     if (!ns_capable(pid_ns->user_ns, CAP_SYS_BOOT))
289         return -EPERM;
290
291     /* For safety, we require "magic" arguments. */
292     if (magic1 != LINUX_REBOOT_MAGIC1 ||
293             (magic2 != LINUX_REBOOT_MAGIC2 &&
294             magic2 != LINUX_REBOOT_MAGIC2A &&
295             magic2 != LINUX_REBOOT_MAGIC2B &&
296             magic2 != LINUX_REBOOT_MAGIC2C))
297         return -EINVAL;
298
299     /*
300      * If pid namespaces are enabled and the current task is in a child
301      * pid_namespace, the command is handled by reboot_pid_ns() which will
302      * call do_exit().
303      */
304     ret = reboot_pid_ns(pid_ns, cmd);
305     if (ret)
306         return ret;
307
308     /* Instead of trying to make the power_off code look like
309      * halt when pm_power_off is not set do it the easy way.
310      */
311     if ((cmd == LINUX_REBOOT_CMD_POWER_OFF) && !pm_power_off)
312         cmd = LINUX_REBOOT_CMD_HALT;
313
314     mutex_lock(&reboot_mutex);
315     switch (cmd) {
        // 我们普通的在命令行执行reboot命令会执行在这里
316     case LINUX_REBOOT_CMD_RESTART:
317         kernel_restart(NULL);
318         break;
319
320     case LINUX_REBOOT_CMD_CAD_ON:
321         C_A_D = 1;
322         break;
323
324     case LINUX_REBOOT_CMD_CAD_OFF:
325         C_A_D = 0;
326         break;
327
328     case LINUX_REBOOT_CMD_HALT:
329         kernel_halt();
330         do_exit(0);
331         panic("cannot halt");
332
333     case LINUX_REBOOT_CMD_POWER_OFF:
334         kernel_power_off();
335         do_exit(0);
336         break;
337
338     case LINUX_REBOOT_CMD_RESTART2:
339         ret = strncpy_from_user(&buffer[0], arg, sizeof(buffer) - 1);
340         if (ret < 0) {
341             ret = -EFAULT;
342             break;
343         }
344         buffer[sizeof(buffer) - 1] = '\0';
345
346         kernel_restart(buffer);
347         break;
348
349 #ifdef CONFIG_KEXEC_CORE
350     case LINUX_REBOOT_CMD_KEXEC:
351         ret = kernel_kexec();
352         break;
353 #endif
354
355 #ifdef CONFIG_HIBERNATION
356     case LINUX_REBOOT_CMD_SW_SUSPEND:
357         ret = hibernate();
358         break;
359 #endif
360
361     default:
362         ret = -EINVAL;
363         break;
364     }
365     mutex_unlock(&reboot_mutex);
366     return ret;
367 }
```

- 3、kernel_restart开始重启系统

```c
206 /**
207  *  kernel_restart - reboot the system
208  *  @cmd: pointer to buffer containing command to execute for restart
209  *      or %NULL
210  *
211  *  Shutdown everything and perform a clean reboot.
212  *  This is not safe to call in interrupt context.
213  */
214 void kernel_restart(char *cmd)
215 {
216     kernel_restart_prepare(cmd);
217     migrate_to_reboot_cpu();
218     syscore_shutdown();
219     if (!cmd)
220         pr_emerg("Restarting system\n");
221     else
222         pr_emerg("Restarting system with command '%s'\n", cmd);
223     kmsg_dump(KMSG_DUMP_RESTART);
    	// 架构相关的重启
224     machine_restart(cmd);
225 }
226 EXPORT_SYMBOL_GPL(kernel_restart);

 68 void kernel_restart_prepare(char *cmd)
 69 {
 70     blocking_notifier_call_chain(&reboot_notifier_list, SYS_RESTART, cmd);
 71     system_state = SYSTEM_RESTART;
 72     usermodehelper_disable();
     	// 调用各个设备的shutdown相关接口
 73     device_shutdown();
 74 }

```





