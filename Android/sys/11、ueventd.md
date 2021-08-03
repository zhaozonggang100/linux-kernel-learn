## 基于Android Q来分析uevent

---

- 1、ueventd如何启动的

  init进程的SecondStageMain阶段中解析init.rc(system/core/rootdir/init.rc)，在其中定义了`start ueventd`，从而进入了ueventd_main(system/core/init/main.cpp)

```bash
# ueventd和init用的一套代码，通过argv[0]来区分
service ueventd /system/bin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0
    shutdown critical
```

- 2、ueventd的启动实际

```
early-init
```