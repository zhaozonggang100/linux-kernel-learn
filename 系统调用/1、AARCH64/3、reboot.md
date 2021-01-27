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

```c++
/*
* Reboot system call: for obvious reasons only root may call it,
* and even root needs to set up some magic numbers in the registers
* so that some mistake won't make this reboot the whole machine.
* You can also set the meaning of the ctrl-alt-del-key here.
*
* reboot doesn't sync: do that yourself before calling this.
*/
SYSCALL_DEFINE4(reboot, int, magic1, int, magic2, unsigned int, cmd,
		void __user *, arg)
{
	struct pid_namespace *pid_ns = task_active_pid_ns(current);
	char buffer[256];
	int ret = 0;

	/* We only trust the superuser with rebooting the system. */
	if (!ns_capable(pid_ns->user_ns, CAP_SYS_BOOT))
		return -EPERM;

	/* For safety, we require "magic" arguments. */
	if (magic1 != LINUX_REBOOT_MAGIC1 ||
			(magic2 != LINUX_REBOOT_MAGIC2 &&
			magic2 != LINUX_REBOOT_MAGIC2A &&
			magic2 != LINUX_REBOOT_MAGIC2B &&
			magic2 != LINUX_REBOOT_MAGIC2C))
		return -EINVAL;

	/*
	* If pid namespaces are enabled and the current task is in a child
	* pid_namespace, the command is handled by reboot_pid_ns() which will
	* call do_exit().
	*/
	ret = reboot_pid_ns(pid_ns, cmd);
	if (ret)
		return ret;

	/* Instead of trying to make the power_off code look like
	* halt when pm_power_off is not set do it the easy way.
	*/
	if ((cmd == LINUX_REBOOT_CMD_POWER_OFF) && !pm_power_off)
		cmd = LINUX_REBOOT_CMD_HALT;

	mutex_lock(&reboot_mutex);
	switch (cmd) {
	// 我们普通的在命令行执行reboot命令会执行在这里
	case LINUX_REBOOT_CMD_RESTART:
		kernel_restart(NULL);
		break;

	case LINUX_REBOOT_CMD_CAD_ON:
		C_A_D = 1;
		break;

	case LINUX_REBOOT_CMD_CAD_OFF:
		C_A_D = 0;
		break;

	case LINUX_REBOOT_CMD_HALT:
		kernel_halt();
		do_exit(0);
		panic("cannot halt");

	case LINUX_REBOOT_CMD_POWER_OFF:
		kernel_power_off();
		do_exit(0);
		break;

	case LINUX_REBOOT_CMD_RESTART2:
		ret = strncpy_from_user(&buffer[0], arg, sizeof(buffer) - 1);
		if (ret < 0) {
			ret = -EFAULT;
			break;
		}
		buffer[sizeof(buffer) - 1] = '\0';

		kernel_restart(buffer);
		break;

#ifdef CONFIG_KEXEC_CORE
	case LINUX_REBOOT_CMD_KEXEC:
		ret = kernel_kexec();
		break;
#endif

#ifdef CONFIG_HIBERNATION
	case LINUX_REBOOT_CMD_SW_SUSPEND:
		ret = hibernate();
		break;
#endif

	default:
		ret = -EINVAL;
		break;
	}
	mutex_unlock(&reboot_mutex);
	return ret;
}
```

- 3、kernel_restart开始重启系统

```c++
/**
*  kernel_restart - reboot the system
*  @cmd: pointer to buffer containing command to execute for restart
*      or %NULL
*
*  Shutdown everything and perform a clean reboot.
*  This is not safe to call in interrupt context.
*/
void kernel_restart(char *cmd)
{
	kernel_restart_prepare(cmd);
	migrate_to_reboot_cpu();
	syscore_shutdown();
	if (!cmd)
		pr_emerg("Restarting system\n");
	else
		pr_emerg("Restarting system with command '%s'\n", cmd);
	kmsg_dump(KMSG_DUMP_RESTART);
	// 架构相关的重启
	machine_restart(cmd);
}
EXPORT_SYMBOL_GPL(kernel_restart);

void kernel_restart_prepare(char *cmd)
{
	blocking_notifier_call_chain(&reboot_notifier_list, SYS_RESTART, cmd);
	system_state = SYSTEM_RESTART;
	usermodehelper_disable();
	// 调用各个设备的shutdown相关接口
	device_shutdown();
}
```