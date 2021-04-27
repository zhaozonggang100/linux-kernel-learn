**ref**
```
https://www.jianshu.com/p/9f978d57c683（Android O启动流程）
http://wp.lijinshan.site/android/13374（Q）
http://www.soolco.com/post/46792_1_1.html(Q)
https://blog.csdn.net/yiranfeng/article/details/103549394（Q）
```
---

# 1、android O

### 1、概要

- 1、kernel启动init进程  

&emsp;&emsp;`start_kernel->rest_init->kernel_init`

&emsp;&emsp; 1、在kernel_init内核线程中先检测ramdisk_execute_command是否设置，ramdisk_execute_command变量是bootloader通过cmdline传递给kernel，`rdinit=xxx`，如果cmdline没有设置，就会在`kernel_init_freeable`中将其设置为`"/init"`,接着判断`/init`是否可执行，不可知性，接着又把ramdisk_execute_command置为NULL.
```c++
// cmdline设置了rdinit的话，解析了cmdline会设置该变量，没有设置就会把它设置为/init
if (!ramdisk_execute_command)
        ramdisk_execute_command = "/init";
// 检测根文件系统是否有init文件
if (sys_access((const char __user *) ramdisk_execute_command, 0) != 0) {
        // 没有则ramdisk_execute_command赋值为NULL，后面接着判断execute_command
        ramdisk_execute_command = NULL;
        prepare_namespace();
}

if (ramdisk_execute_command) {
    ret = run_init_process(ramdisk_execute_command);
    if (!ret)
            return 0;
    pr_err("Failed to execute %s (error %d)\n",
            ramdisk_execute_command, ret);
}
```  

&emsp;&emsp; 2、如果第一步没有设置ramdisk_execute_command变量，接着检测execute_command变量是否设置，该变量是由bootloader通过init=xxx设置的，找到则执行对应的bin文件
```c++
/*
 * We try each of these until one succeeds.
 *
 * The Bourne shell can be used instead of init if we are
 * trying to recover a really broken machine.
 */
if (execute_command) {
    ret = run_init_process(execute_command);
    if (!ret)
            return 0;
    panic("Requested init %s failed (error %d).",
            execute_command, ret);
}
```

&emsp;&emsp; 3、都没设置，接着会去根文件系统下依次查找/sbin/init、/etc/init、/bin/init、/bin/sh，如果都找不到认为没有合适的init进程执行。
```c++
if (!try_to_run_init_process("/sbin/init") ||
    !try_to_run_init_process("/etc/init") ||
    !try_to_run_init_process("/bin/init") ||
    !try_to_run_init_process("/bin/sh"))
            return 0;

panic("No working init found.  Try passing init= option to kernel. "
      "See Linux Documentation/init.txt for guidance.");
```

### 2、源码文件

```bash
# file : system/core/init/
total 692
-rw-rw-r-- 1 akon akon   4267 Dec  1 12:49 Android.bp
-rw-rw-r-- 1 akon akon   2129 Dec  1 12:49 Android.mk
-rw-rw-r-- 1 akon akon      0 Dec  1 12:49 MODULE_LICENSE_APACHE2
-rw-rw-r-- 1 akon akon  10695 Dec  1 12:49 NOTICE
-rw-rw-r-- 1 akon akon  26649 Dec  1 12:49 README.md
-rw-rw-r-- 1 akon akon  11355 Dec  1 12:49 action.cpp
-rw-rw-r-- 1 akon akon   4387 Dec  1 12:49 action.h
-rw-rw-r-- 1 akon akon   6045 Dec  1 12:49 bootchart.cpp
-rw-rw-r-- 1 akon akon    868 Dec  1 12:49 bootchart.h
-rwxrwxr-x 1 akon akon  34219 Dec  1 12:49 builtins.cpp
-rw-rw-r-- 1 akon akon   1095 Dec  1 12:49 builtins.h
-rw-rw-r-- 1 akon akon   6414 Dec  1 12:49 capabilities.cpp
-rw-rw-r-- 1 akon akon   1238 Dec  1 12:49 capabilities.h
-rwxrwxr-x 1 akon akon   5471 Dec  1 12:49 compare-bootcharts.py
-rw-rw-r-- 1 akon akon   4483 Dec  1 12:49 descriptors.cpp
-rw-rw-r-- 1 akon akon   2511 Dec  1 12:49 descriptors.h
-rw-rw-r-- 1 akon akon  18158 Dec  1 12:49 devices.cpp
-rw-rw-r-- 1 akon akon   4216 Dec  1 12:49 devices.h
-rw-rw-r-- 1 akon akon  12371 Dec  1 12:49 devices_test.cpp
-rw-rw-r-- 1 akon akon   4025 Dec  1 12:49 firmware_handler.cpp
-rw-rw-r-- 1 akon akon    849 Dec  1 12:49 firmware_handler.h
-rwxrwxr-x 1 akon akon    623 Dec  1 12:49 grab-bootchart.sh
-rw-rw-r-- 1 akon akon   1765 Dec  1 12:49 import_parser.cpp
-rw-rw-r-- 1 akon akon   1360 Dec  1 12:49 import_parser.h
-rwxrwxr-x 1 akon akon  43303 Dec  1 12:49 init.cpp
-rw-rw-r-- 1 akon akon   1581 Dec  1 12:49 init.h
-rw-rw-r-- 1 akon akon  19022 Dec  1 12:49 init_first_stage.cpp
-rw-rw-r-- 1 akon akon    832 Dec  1 12:49 init_first_stage.h
-rw-rw-r-- 1 akon akon   5509 Dec  1 12:49 init_parser.cpp
-rw-rw-r-- 1 akon akon   4289 Dec  1 12:49 init_parser.h
-rw-rw-r-- 1 akon akon   4763 Dec  1 12:49 init_parser_test.cpp
-rw-rw-r-- 1 akon akon   6451 Dec  1 12:49 init_test.cpp
-rw-rw-r-- 1 akon akon   3590 Dec  1 12:49 keychords.cpp
-rw-rw-r-- 1 akon akon    853 Dec  1 12:49 keychords.h
-rw-rw-r-- 1 akon akon   2857 Dec  1 12:49 keyword_map.h
-rw-rw-r-- 1 akon akon   1881 Dec  1 12:49 log.cpp
-rw-rw-r-- 1 akon akon    898 Dec  1 12:49 log.h
drwxrwxr-x 2 akon akon   4096 Dec  1 12:49 parser
-rw-rw-r-- 1 akon akon   2821 Dec  1 12:49 parser.cpp
-rw-rw-r-- 1 akon akon    952 Dec  1 12:49 parser.h
-rwxrwxr-x 1 akon akon  16159 Dec  1 12:49 perfboot.py
-rw-rw-r-- 1 akon akon  26354 Dec  1 12:49 property_service.cpp
-rw-rw-r-- 1 akon akon   1235 Dec  1 12:49 property_service.h
-rw-rw-r-- 1 akon akon   1864 Dec  1 12:49 property_service_test.cpp
-rw-rw-r-- 1 akon akon  20294 Dec  1 12:49 reboot.cpp
-rw-rw-r-- 1 akon akon   1350 Dec  1 12:49 reboot.h
-rw-rw-r-- 1 akon akon  41679 Dec  1 12:49 service.cpp
-rw-rw-r-- 1 akon akon  10191 Dec  1 12:49 service.h
-rw-rw-r-- 1 akon akon   2837 Dec  1 12:49 service_test.cpp
-rw-rw-r-- 1 akon akon   1965 Dec  1 12:49 signal_handler.cpp
-rw-rw-r-- 1 akon akon    810 Dec  1 12:49 signal_handler.h
-rw-rw-r-- 1 akon akon 106082 Feb  1 15:22 tags
drwxrwxr-x 2 akon akon   4096 Dec  1 12:49 test_service
-rw-rw-r-- 1 akon akon   1011 Dec  1 12:49 uevent.h
-rw-rw-r-- 1 akon akon   7060 Dec  1 12:49 uevent_listener.cpp
-rw-rw-r-- 1 akon akon   1854 Dec  1 12:49 uevent_listener.h
-rw-rw-r-- 1 akon akon  11255 Dec  1 12:49 ueventd.cpp
-rw-rw-r-- 1 akon akon    805 Dec  1 12:49 ueventd.h
-rw-rw-r-- 1 akon akon   4641 Dec  1 12:49 ueventd_parser.cpp
-rw-rw-r-- 1 akon akon   1738 Dec  1 12:49 ueventd_parser.h
-rw-rw-r-- 1 akon akon   4046 Dec  1 12:49 ueventd_test.cpp
-rw-rw-r-- 1 akon akon  13350 Dec  1 12:49 util.cpp
-rw-rw-r-- 1 akon akon   2634 Dec  1 12:49 util.h
-rw-rw-r-- 1 akon akon   6504 Dec  1 12:49 util_test.cpp
-rw-rw-r-- 1 akon akon   2281 Dec  1 12:49 watchdogd.cpp
-rw-rw-r-- 1 akon akon    811 Dec  1 12:49 watchdogd.h
```

### 3、init进程入口

**file**：`system/core/init/init.cpp`  
**main**函数主要功能：
```c++
/* 01. 创建文件系统目录并挂载相关的文件系统 */
/* 02. 屏蔽标准的输入输出/初始化内核log系统 */
/* 03. 初始化属性域 */
/* 04. 完成SELinux相关工作 */·
/* 05. 重新设置属性 */
/* 06. 创建epoll句柄 */
/* 07. 装载子进程信号处理器 */
/* 08. 设置默认系统属性 */
/* 09. 启动配置属性的服务端 */
/* 10. 匹配命令和函数之间的对应关系 */
int main(int argc, char** argv) {
    // 检测启动文件的名字，uevented、watchdogd也是使用init.cpp编译的
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }    

    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }    

    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }    

    // 设置环境变量
    add_environment("PATH", _PATH_DEFPATH);

    bool is_first_stage = (getenv("INIT_SECOND_STAGE") == nullptr);

    if (is_first_stage) {
        boot_clock::time_point start_time = boot_clock::now();

        // Clear the umask.
        // 设置文件属性为0777
        umask(0);

        // Get the basic filesystem setup we need put together in the initramdisk
        // on / and then we'll let the rc file figure out the rest.
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        #define MAKE_STR(x) __STRING(x)
        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
        // Don't expose the raw commandline to unprivileged processes.
        chmod("/proc/cmdline", 0440);
        gid_t groups[] = { AID_READPROC };
        setgroups(arraysize(groups), groups);
        mount("sysfs", "/sys", "sysfs", 0, NULL);
        mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);
        mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11));
        mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8)); 
        mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9)); 

        // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
        // talk to the outside world...
        InitKernelLogging(argv);

        LOG(ERROR) << "init first stage started!";

        if (!DoFirstStageMount()) {
            LOG(ERROR) << "Failed to mount required partitions early ...";
            panic();
        }

        SetInitAvbVersionInRecovery();

        // Set up SELinux, loading the SELinux policy.
        selinux_initialize(true);

        // We're in the kernel domain, so re-exec init to transition to the init domain now
        // that the SELinux policy has been loaded.
        // 按照selinux policy要求，重新设置init文件属性
        if (selinux_android_restorecon("/init", 0) == -1) {
            PLOG(ERROR) << "restorecon failed";
            security_failure();
        }

        setenv("INIT_SECOND_STAGE", "true", 1);

        static constexpr uint32_t kNanosecondsPerMillisecond = 1e6;
        uint64_t start_ms = start_time.time_since_epoch().count() / kNanosecondsPerMillisecond;
        setenv("INIT_STARTED_AT", std::to_string(start_ms).c_str(), 1);

        char* path = argv[0];
        char* args[] = { path, nullptr };
        execv(path, args);

        // execv() only returns if an error happened, in which case we
        // panic and never fall through this conditional.
        PLOG(ERROR) << "execv(\"" << path << "\") failed";
        security_failure();
    }

    // At this point we're in the second stage of init.
    InitKernelLogging(argv);
    LOG(INFO) << "init second stage started!";

    // Set up a session keyring that all processes will have access to. It
    // will hold things like FBE encryption keys. No process should override
    // its session keyring.
    keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1);

    // Indicate that booting is in progress to background fw loaders, etc.
    // 创建初始化标志
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
    // 初始化Android的属性系统
    property_init();

    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    // 解析DT和命令行中的kernel启动参数
    process_kernel_dt();
    process_kernel_cmdline();

    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    // 设置系统属性
    export_kernel_boot_props();

    // Make the time that init started available for bootstat to log.
    property_set("ro.boottime.init", getenv("INIT_STARTED_AT"));
    property_set("ro.boottime.init.selinux", getenv("INIT_SELINUX_TOOK"));

    // Set libavb version for Framework-only OTA match in Treble build.
    const char* avb_version = getenv("INIT_AVB_VERSION");
    if (avb_version) property_set("ro.boot.avb_version", avb_version);

    // Set memcg property based on kernel cmdline argument
    bool memcg_enabled = android::base::GetBoolProperty("ro.boot.memcg",false);
    if (memcg_enabled) {
       // root memory control cgroup
       mkdir("/dev/memcg", 0700);
       chown("/dev/memcg",AID_ROOT,AID_SYSTEM);
       mount("none", "/dev/memcg", "cgroup", 0, "memory");
       // app mem cgroups, used by activity manager, lmkd and zygote
       mkdir("/dev/memcg/apps/",0755);
       chown("/dev/memcg/apps/",AID_SYSTEM,AID_SYSTEM);
       mkdir("/dev/memcg/system",0550);
       chown("/dev/memcg/system",AID_SYSTEM,AID_SYSTEM);
    }

    // Clean up our environment.
    unsetenv("INIT_SECOND_STAGE");
    unsetenv("INIT_STARTED_AT");
    unsetenv("INIT_SELINUX_TOOK");
    unsetenv("INIT_AVB_VERSION");

    // Now set up SELinux for second stage.
    // 调用selinux_initialize函数启动SELinux
    selinux_initialize(false);
    selinux_restore_context();

    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        PLOG(ERROR) << "epoll_create1 failed";
        exit(1);
    }
    signal_handler_init();

    property_load_boot_defaults();
    export_oem_lock_status();
    start_property_service();
    set_usb_controller();

    const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);

    ActionManager& am = ActionManager::GetInstance();
    ServiceManager& sm = ServiceManager::GetInstance();
    Parser& parser = Parser::GetInstance();

    parser.AddSectionParser("service", std::make_unique<ServiceParser>(&sm));
    parser.AddSectionParser("on", std::make_unique<ActionParser>(&am));
    parser.AddSectionParser("import", std::make_unique<ImportParser>(&parser));
    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/init.rc");
        parser.set_is_system_etc_init_loaded(
                parser.ParseConfig("/system/etc/init"));
        parser.set_is_vendor_etc_init_loaded(
                parser.ParseConfig("/vendor/etc/init"));
        parser.set_is_odm_etc_init_loaded(parser.ParseConfig("/odm/etc/init"));
    } else {
        parser.ParseConfig(bootscript);
        parser.set_is_system_etc_init_loaded(true);
        parser.set_is_vendor_etc_init_loaded(true);
        parser.set_is_odm_etc_init_loaded(true);
    }

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) DumpState();

    am.QueueEventTrigger("early-init");

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
    am.QueueBuiltinAction(set_kptr_restrict_action, "set_kptr_restrict");
    am.QueueBuiltinAction(keychord_init_action, "keychord_init");
    am.QueueBuiltinAction(console_init_action, "console_init");
    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    // init进程最后进入了无限循环中
    while (true) {
        // By default, sleep until something happens.
        int epoll_timeout_ms = -1;

        if (do_shutdown && !shutting_down) {
            do_shutdown = false;
            if (HandlePowerctlMessage(shutdown_command)) {
                shutting_down = true;
            }
        }

        // 调用函数execute_one_command来检查action_queue列表是否为空。如果不为空的话，那么init进程就将保存在列表头部中的action移除，并且执行这个被移除的action。由于前面我们将一个名称为"console_init"的action添加到action_queue列表中，因此，在这个无线循环中，这个action就会被执行，即console_init_action函数会被调用
        if (!(waiting_for_prop || sm.IsWaitingForExec())) {
            am.ExecuteOneCommand();
        }

        // 调用函数restart_processes来检查系统中是否有进程需要重启。在启动脚本init.rc中，我们可以指定一个进程在退出之后会自动重启。在这种情况下，函数restart_processes就会检查是否存在需要重新启动的进程，如果存在的话，那么就将它重新启动起来
        if (!(waiting_for_prop || sm.IsWaitingForExec())) {
            if (!shutting_down) restart_processes();

            // If there's a process that needs restarting, wake up in time for that.
            if (process_needs_restart_at != 0) {
                epoll_timeout_ms = (process_needs_restart_at - time(nullptr)) * 1000;
                if (epoll_timeout_ms < 0) epoll_timeout_ms = 0;
            }

            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) epoll_timeout_ms = 0;
        }

        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
        if (nr == -1) {
            PLOG(ERROR) << "epoll_wait failed";
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }

    return 0;
}

}  // namespace init
}  // namespace android

// 整个init进程的入口
int main(int argc, char** argv) {
    android::init::main(argc, argv);
}
```

# 2、Android Q

### 1、入口main

Android Q上init.cpp的main函数会分多个阶段调用

```c++
//  android/system/core/init/main.cpp
int main(int argc, char** argv) {
#if __has_feature(address_sanitizer)
    __asan_set_error_report_callback(AsanReportCallback);
#endif

    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap function_map;

            return SubcontextMain(argc, argv, &function_map);
        }

        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }

        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
    }

    // 第一次调用init，会进行分区的挂载
    return FirstStageMain(argc, argv);
}
```

- 1、第一阶段调用`FirstStageMain`完成分区挂载的动作

```c++
// android/system/core/init/first_stage_init.cpp
int FirstStageMain(int argc, char** argv) {
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    boot_clock::time_point start_time = boot_clock::now();

    std::vector<std::pair<std::string, int>> errors;
#define CHECKCALL(x) \
    if (x != 0) errors.emplace_back(#x " failed", errno);

    // Clear the umask.
    umask(0);

    CHECKCALL(clearenv());
    CHECKCALL(setenv("PATH", _PATH_DEFPATH, 1));
    // Get the basic filesystem setup we need put together in the initramdisk
    // on / and then we'll let the rc file figure out the rest.
    CHECKCALL(mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755"));
    CHECKCALL(mkdir("/dev/pts", 0755));
    CHECKCALL(mkdir("/dev/socket", 0755));
    CHECKCALL(mount("devpts", "/dev/pts", "devpts", 0, NULL));
#define MAKE_STR(x) __STRING(x)
    #ifdef ENABLE_EARLY_SERVICES
     if (0 == access("/proc/cmdline", F_OK))
     {
     CHECKCALL(mount("proc", "/proc", "proc", MS_REMOUNT, "hidepid=2,gid=" MAKE_STR(AID_READPROC)));
     }
     else
     {
     CHECKCALL(mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC)));
     }
    #else
    CHECKCALL(mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC)));
    #endif
#undef MAKE_STR
    // Don't expose the raw commandline to unprivileged processes.
    CHECKCALL(chmod("/proc/cmdline", 0440));
    gid_t groups[] = {AID_READPROC};
    CHECKCALL(setgroups(arraysize(groups), groups));
    #ifdef ENABLE_EARLY_SERVICES
    mount("sysfs", "/sys", "sysfs", 0, NULL);
    mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);
    #else
    CHECKCALL(mount("sysfs", "/sys", "sysfs", 0, NULL));
    CHECKCALL(mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL));
    #endif
    CHECKCALL(mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11)));

    if constexpr (WORLD_WRITABLE_KMSG) {
        CHECKCALL(mknod("/dev/kmsg_debug", S_IFCHR | 0622, makedev(1, 11)));
    }

    CHECKCALL(mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8)));
    CHECKCALL(mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9)));

    // This is needed for log wrapper, which gets called before ueventd runs.
    CHECKCALL(mknod("/dev/ptmx", S_IFCHR | 0666, makedev(5, 2)));
    CHECKCALL(mknod("/dev/null", S_IFCHR | 0666, makedev(1, 3)));

    // These below mounts are done in first stage init so that first stage mount can mount
    // subdirectories of /mnt/{vendor,product}/.  Other mounts, not required by first stage mount,
    // should be done in rc files.
    // Mount staging areas for devices managed by vold
    // See storage config details at http://source.android.com/devices/storage/
    CHECKCALL(mount("tmpfs", "/mnt", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=1000"));
    // /mnt/vendor is used to mount vendor-specific partitions that can not be
    // part of the vendor partition, e.g. because they are mounted read-write.
    CHECKCALL(mkdir("/mnt/vendor", 0755));
    // /mnt/product is used to mount product-specific partitions that can not be
    // part of the product partition, e.g. because they are mounted read-write.
    CHECKCALL(mkdir("/mnt/product", 0755));

    // /apex is used to mount APEXes
    CHECKCALL(mount("tmpfs", "/apex", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=0"));

    // /debug_ramdisk is used to preserve additional files from the debug ramdisk
    CHECKCALL(mount("tmpfs", "/debug_ramdisk", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=0"));
#undef CHECKCALL

    SetStdioToDevNull(argv);
    // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
    // talk to the outside world...
    InitKernelLogging(argv);

    if (!errors.empty()) {
        for (const auto& [error_string, error_errno] : errors) {
            LOG(ERROR) << error_string << " " << strerror(error_errno);
        }
        LOG(FATAL) << "Init encountered errors starting first stage, aborting";
    }

    LOG(INFO) << "init first stage started!";

    auto old_root_dir = std::unique_ptr<DIR, decltype(&closedir)>{opendir("/"), closedir};
    if (!old_root_dir) {
        PLOG(ERROR) << "Could not opendir(\"/\"), not freeing ramdisk";
    }

    struct stat old_root_info;
    if (stat("/", &old_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        old_root_dir.reset();
    }

    if (ForceNormalBoot()) {
        mkdir("/first_stage_ramdisk", 0755);
        // SwitchRoot() must be called with a mount point as the target, so we bind mount the
        // target directory to itself here.
        if (mount("/first_stage_ramdisk", "/first_stage_ramdisk", nullptr, MS_BIND, nullptr) != 0) {
            LOG(FATAL) << "Could not bind mount /first_stage_ramdisk to itself";
        }
        SwitchRoot("/first_stage_ramdisk");
    }

    // If this file is present, the second-stage init will use a userdebug sepolicy
    // and load adb_debug.prop to allow adb root, if the device is unlocked.
    if (access("/force_debuggable", F_OK) == 0) {
        std::error_code ec;  // to invoke the overloaded copy_file() that won't throw.
        if (!fs::copy_file("/adb_debug.prop", kDebugRamdiskProp, ec) ||
            !fs::copy_file("/userdebug_plat_sepolicy.cil", kDebugRamdiskSEPolicy, ec)) {
            LOG(ERROR) << "Failed to setup debug ramdisk";
        } else {
            // setenv for second-stage init to read above kDebugRamdisk* files.
            setenv("INIT_FORCE_DEBUGGABLE", "true", 1);
        }
    }

    // 内部会挂载Q的逻辑分区
    if (!DoFirstStageMount()) {
        LOG(FATAL) << "Failed to mount required partitions early ...";
    }

    struct stat new_root_info;
    if (stat("/", &new_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        old_root_dir.reset();
    }

    if (old_root_dir && old_root_info.st_dev != new_root_info.st_dev) {
        FreeRamdisk(old_root_dir.get(), old_root_info.st_dev);
    }

    SetInitAvbVersionInRecovery();

    static constexpr uint32_t kNanosecondsPerMillisecond = 1e6;
    uint64_t start_ms = start_time.time_since_epoch().count() / kNanosecondsPerMillisecond;
    setenv("INIT_STARTED_AT", std::to_string(start_ms).c_str(), 1);

    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    // 第二次执行init，参数是selinux_setup
    execv(path, const_cast<char**>(args));

    // execv() only returns if an error happened, in which case we
    // panic and never fall through this conditional.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
}
```

- 2、第二阶段会在`FirstStageMain`中再次执行init，传参为`selinux_setup`，完成selinux的初始化

- 3、第三阶段会在`SetupSelinux`中第三次执行init，参数为`second_stage`，完成信号量、各种服务的初始化，最后进入`while 1`循环

### 2、super分区挂载过程

init--->FirstStageMain()--->DoFirstStageMount()--->FirstStageMount::DoFirstStageMount()

- 1、DoFirstStageMount(public)，从/vendor/etc/fstab.qcom构建FirstStageMount类的对象，并调用对象的DoFirstStageMount方法

```c++
// file：android/system/core/init/first_stage_mount.cpp
// 这个函数是public的，不属于类
bool DoFirstStageMount() {
    // Skips first stage mount if we're in recovery mode.
    if (IsRecoveryMode()) {
        LOG(INFO) << "First stage mount skipped (recovery mode)";
        return true;
    }

    // 调用create返回FirstStageMount的类对象（实际是通过fstab类型强转）
    // create函数内部会读取Android device tree（/proc/device-tree/firmware/android）的配置返回不同的对象
    //      get_android_dt_dir()函数返回：/proc/device-tree/firmware/android
    std::unique_ptr<FirstStageMount> handle = FirstStageMount::Create();
    if (!handle) {
        LOG(ERROR) << "Failed to create FirstStageMount";
        return false;
    }

    // 调用FirstStageMount类的DoFirstStageMount方法
    return handle->DoFirstStageMount();
}

std::unique_ptr<FirstStageMount> FirstStageMount::Create() {
    auto fstab = ReadFirstStageFstab();
    if (IsDtVbmetaCompatible(fstab)) {
        // 从此处返回
        // class FirstStageMountVBootV2 : public FirstStageMount
        return std::make_unique<FirstStageMountVBootV2>(std::move(fstab));
    } else {
        return std::make_unique<FirstStageMountVBootV1>(std::move(fstab));
    }
}

static Fstab ReadFirstStageFstab() {
    Fstab fstab;
    if (!ReadFstabFromDt(&fstab)) { // 从/proc/device-tree/firmware/android读取fstab失败
        if (ReadDefaultFstab(&fstab)) { // 从/vendor/etc/fstab.qcom读取fstab条目
            fstab.erase(std::remove_if(fstab.begin(), fstab.end(),
                                       [](const auto& entry) {
                                           return !entry.fs_mgr_flags.first_stage_mount;
                                       }),
                        fstab.end());
        } else {
            LOG(INFO) << "Failed to fstab for first stage mount";
        }
    }
    return fstab;
}
```

- 2、FirstStageMount::DoFirstStageMount()--->FirstStageMount::InitDevices

```c++
// file：android/system/core/init/first_stage_mount.cpp
bool FirstStageMount::DoFirstStageMount() {
    // 判断：两个条件都是真
    //      1、内核是否使能dm-linner
    //      2、上一步的fstab条目是否为空
    if (!IsDmLinearEnabled() && fstab_.empty()) {
        // Nothing to mount.
        LOG(INFO) << "First stage mount skipped (missing/incompatible/empty fstab in device tree)";
        return true;
    }

    // 1. 初始化device-mapper uevent，super分区添加到动态分区列表
    if (!InitDevices()) return false;

    // 2. 
    if (!CreateLogicalPartitions()) return false;

    // 3
    if (!MountPartitions()) return false;

    return true;
}

// --------------------------> 1
// FirstStageMount::InitDevices()--->FirstStageMount::GetDmLinearMetadataDevice()--->FirstStageMountVBootV2::GetDmVerityDevices()--->FirstStageMount::InitRequiredDevices()
bool FirstStageMount::InitDevices() {
    // i: FirstStageMount->required_devices_partition_names_ = "super"
    // ii: 
    // iii: 监听/devices/virtual/misc/device-mapper/uevent
    return GetDmLinearMetadataDevice() && GetDmVerityDevices() && InitRequiredDevices();
}

// 获取dm设备对应的物理分区：Q中定义的是：super
bool FirstStageMount::GetDmLinearMetadataDevice() {
    // Add any additional devices required for dm-linear mappings.
    if (!IsDmLinearEnabled()) { // return true：1
        return true;
    }

    // super_partition_name_使用了默认的 ： "super"
    required_devices_partition_names_.emplace(super_partition_name_);
    return true;
}

bool FirstStageMountVBootV2::GetDmVerityDevices() {
    need_dm_verity_ = false;

    std::set<std::string> logical_partitions;

    // fstab_rec->blk_device has A/B suffix.
    // 遍历fstab
    for (const auto& fstab_entry : fstab_) {
        if (fstab_entry.fs_mgr_flags.avb) {
            need_dm_verity_ = true;
        }
        // fs_mgr_flags中标记分区是否是逻辑分区
        if (fstab_entry.fs_mgr_flags.logical) { // 逻辑分区
            // Don't try to find logical partitions via uevent regeneration.
            logical_partitions.emplace(basename(fstab_entry.blk_device.c_str()));
        } else { // 物理分区
            required_devices_partition_names_.emplace(basename(fstab_entry.blk_device.c_str()));
        }
    }

    // Any partitions needed for verifying the partitions used in first stage mount, e.g. vbmeta
    // must be provided as vbmeta_partitions.
    // vbmeta分区：vbmeta,boot,system,vendor,dtboa
    if (need_dm_verity_) {
        if (vbmeta_partitions_.empty()) {
            LOG(ERROR) << "Missing vbmeta partitions";
            return false;
        }
        std::string ab_suffix = fs_mgr_get_slot_suffix();
        for (const auto& partition : vbmeta_partitions_) {
            std::string partition_name = partition + ab_suffix;
            if (logical_partitions.count(partition_name)) {  // 逻辑分区跳过
                continue;
            }
            // required_devices_partition_names_ is of type std::set so it's not an issue
            // to emplace a partition twice. e.g., /vendor might be in both places:
            //   - device_tree_vbmeta_parts_ = "vbmeta,boot,system,vendor"
            //   - mount_fstab_recs_: /vendor_a
            required_devices_partition_names_.emplace(partition_name);
        }
    }
    return true;
}

// Creates devices with uevent->partition_name matching one in the member variable
// required_devices_partition_names_. Found partitions will then be removed from it
// for the subsequent member function to check which devices are NOT created.
bool FirstStageMount::InitRequiredDevices() {
    if (required_devices_partition_names_.empty()) {
        return true;
    }

    if (IsDmLinearEnabled() || need_dm_verity_) {
        if (!InitDeviceMapper()) {
            return false;
        }
    }

    // 递归扫描/sys目录下的uevent文件，如果对应的块设备驱动没有创建uevent文件则删除分区从required_devices_partition_names_
    auto uevent_callback = [this](const Uevent& uevent) { return UeventCallback(uevent); };
    uevent_listener_.RegenerateUevents(uevent_callback);

    // UeventCallback() will remove found partitions from required_devices_partition_names_.
    // So if it isn't empty here, it means some partitions are not found.
    if (!required_devices_partition_names_.empty()) {
        LOG(INFO) << __PRETTY_FUNCTION__
                  << ": partition(s) not found in /sys, waiting for their uevent(s): "
                  << android::base::Join(required_devices_partition_names_, ", ");
        Timer t;
        uevent_listener_.Poll(uevent_callback, 10s);
        LOG(INFO) << "Wait for partitions returned after " << t;
    }

    if (!required_devices_partition_names_.empty()) {
        LOG(ERROR) << __PRETTY_FUNCTION__ << ": partition(s) not found after polling timeout: "
                   << android::base::Join(required_devices_partition_names_, ", ");
        return false;
    }

    return true;
}

// --------------------------> 2
bool FirstStageMount::CreateLogicalPartitions() {
    if (!IsDmLinearEnabled()) {
        return true;
    }
    if (lp_metadata_partition_.empty()) {
        LOG(ERROR) << "Could not locate logical partition tables in partition "
                   << super_partition_name_;
        return false;
    }

    auto metadata = android::fs_mgr::ReadCurrentMetadata(lp_metadata_partition_);
    if (!metadata) {
        LOG(ERROR) << "Could not read logical partition metadata from " << lp_metadata_partition_;
        return false;
    }
    if (!InitDmLinearBackingDevices(*metadata.get())) {
        return false;
    }
    return android::fs_mgr::CreateLogicalPartitions(*metadata.get(), lp_metadata_partition_);
}
```