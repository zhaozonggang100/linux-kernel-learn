**ref**

```
https://www.ibm.com/developerworks/cn/linux/l-devmapper/index.html
https://blog.csdn.net/u012932409/article/details/105075851
https://source.android.google.cn/devices/tech/ota/dynamic_partitions/implement?hl=nl#fstab-changes
```

---

### 1、概述

Android 10支持了动态分区，这是一种可以通过无线下载 (OTA) 更新来创建、销毁分区或调整分区大小的用户空间分区系统。系统为设备分配一个 super 分区，其中的子分区可动态地调整大小。单个分区映像不再需要为将来的 OTA 预留空间，super 中剩余的可用空间可用于所有动态分区。

动态分区是使用 Linux 内核中的 `dm-linear`、`device-mapper` 模块实现的。super 分区中包含列出 super 内每个动态分区的名称和块范围的元数据。在第一阶段 init 执行期间，系统会解析和验证这些元数据，并创建`虚拟块设备`来表示每个动态分区。

应用 OTA 时，系统会根据需要自动创建/删除动态分区，或者调整动态分区的大小。若是 A/B 设备，将存在两个元数据副本，而更改仅会应用到表示目标槽位的副本。 由于动态分区是在用户空间中实现的，因此引导加载程序所需的分区不能是动态的。例如，引导加载程序会读取 boot、dtbo 和 vbmeta，因此这些分区必须仍保持为物理分区。

### 2、super支持的动态分区

- system

- vendor

- odm

- userdata？？？

### 3、super分区对齐

如果 super 分区未正确对齐，`device-mapper` 模块的运行效率可能会降低。super 分区`必须`与最小 I/O 请求大小保持一致，该大小由块层决定。默认情况下，编译系统（通过生成 super 分区映像的 lpmake）认为对每个动态分区应用 1 MiB 的对齐就已经足够。不过，供应商应确保 super 分区正确对齐。

- 查看当前机器的super分区对齐信息

```bash
# 1、确定块设备的最小请求大小
# ls -l /dev/block/by-name/super
lrwxrwxrwx 1 root root 16 1970-04-05 01:41 /dev/block/by-name/super -> /dev/block/sda17
# cat /sys/block/sda/queue/minimum_io_size
786432

# 2、验证 super 分区的对齐，下面结果必须是0
# cat /sys/block/sda/sda17/alignment_offset
```

### 4、查看系统是否开启动态分区配置

```bash
# file：android/device/google/cuttlefish/shared/phone/device.mk
TARGET_USE_DYNAMIC_PARTITIONS ?= true
ifeq ($(TARGET_USE_DYNAMIC_PARTITIONS),true)
  PRODUCT_USE_DYNAMIC_PARTITIONS := true
  TARGET_BUILD_SYSTEM_ROOT_IMAGE := false
else
  TARGET_BUILD_SYSTEM_ROOT_IMAGE ?= true
endif
```

### 5、动态分区配置

在A/B设备上，如果super.img的大小超过动态大小的一半，则会编译报错  

- 动态分区大小配置  

```bash
# file：android/device/mega/x01_8155r/BoardConfig.mk
ifeq ($(ENABLE_AB), true)
  BOARD_SUPER_PARTITION_SIZE := 12884901888
else
  BOARD_SUPER_PARTITION_SIZE := 5318967296
endif
```

- 动态分区支持分区列表  

```bash
# file：android/device/mega/x01_8155r/BoardConfig.mk
BOARD_QTI_DYNAMIC_PARTITIONS_PARTITION_LIST := system vendor
```