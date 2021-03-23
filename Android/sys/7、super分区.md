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

### 6、创建一个动态分区

- 1、修改分区表，添加自己动态分区对应的物理分区  

- 2、生成动态分区镜像

```bash
#!/bin/bash

# xxx.img生成过程
# 1、dd if=/dev/zero of=xxx.ext4 count=xxx
# 2、mkfs.ext4 xxx.ext4
# 3、img2simg xxx.ext4 xxx.img  将ext4格式的文件转换为sparse（https://blog.csdn.net/js_wawayu/article/details/52420255）

# lpmake源码：android/system/core/fs_mgr/liblp/
lpmake --metadata-size 65536 \
        --super-name super_liauto \
        --metadata-slots 3 \ 
        --device super_liauto:1073741824 \
        --group qti_dynamic_partitions_liauto:1073741824 \
        --partition li_1:none:52428800:qti_dynamic_partitions_liauto \
        --image li_1=./li_1.img \
        --partition li_2:none:52428800:qti_dynamic_partitions_liauto \
        --image li_2=./li_2.img \
        --sparse \
        --output ./super-liauto.img
```

- 3、

### 7、super镜像的生成过程

ref：`https://www.codenong.com/cs106478036/`

- makefile：`android/build/make/core/Makefile`

```bash
INSTALLED_SUPERIMAGE_TARGET := $(PRODUCT_OUT)/super.img
INSTALLED_SUPERIMAGE_DEPENDENCIES := $(LPMAKE) $(BUILD_SUPER_IMAGE) \
    $(foreach p, $(BOARD_SUPER_PARTITION_PARTITION_LIST), $(INSTALLED_$(call to-upper,$(p))IMAGE_TARGET))

# Build super.img by using $(INSTALLED_*IMAGE_TARGET) to $(1)
# $(1): built image path
# $(2): misc_info.txt path; its contents should match expectation of build_super_image.py
define build-superimage-target
  mkdir -p $(dir $(2))
  rm -rf $(2)
  $(call dump-super-image-info,$(2))
  $(foreach p,$(BOARD_SUPER_PARTITION_PARTITION_LIST), \
    echo "$(p)_image=$(INSTALLED_$(call to-upper,$(p))IMAGE_TARGET)" >> $(2);)
  mkdir -p $(dir $(1))
  # 下面就是使用build_super_image.py工具，misc_info.txt作为metadata参数，生成super.img（sparse）镜像
  PATH=$(dir $(LPMAKE)):$$PATH \
    $(BUILD_SUPER_IMAGE) -v $(2) $(1)
endef

# Build $(PRODUCT_OUT)/super.img without dependencies.
.PHONY: superimage-nodeps supernod
superimage-nodeps supernod: intermediates :=
superimage-nodeps supernod: | $(INSTALLED_SUPERIMAGE_DEPENDENCIES)
        $(call pretty,"make $(INSTALLED_SUPERIMAGE_TARGET): ignoring dependencies")
        $(call build-superimage-target,$(INSTALLED_SUPERIMAGE_TARGET),\
          $(call intermediates-dir-for,PACKAGING,superimage-nodeps)/misc_info.txt)
```

- make supernod : 生成super.img镜像，日志如下

```bash
[100% 5001/5001] make out/target/product/HU_8155F_X01/super.img: ignoring dependencies
2021-03-22 07:18:02 - build_super_image.py - INFO    : Building super image from info dict...
2021-03-22 07:18:02 - sparse_img.py - INFO    : Total of 375422 4096-byte output blocks in 19 input chunks.
2021-03-22 07:18:02 - sparse_img.py - INFO    : Total of 133069 4096-byte output blocks in 13 input chunks.
2021-03-22 07:18:02 - common.py - INFO    :   Running: "lpmake --metadata-size 65536 --super-name super --metadata-slots 3 --device super:12884901888 --group qti_dynamic_partitions_a:5314772992 --group qti_dynamic_partitions_b:5314772992 --partition system_a:readonly:1537728512:qti_dynamic_partitions_a --image system_a=out/target/product/HU_8155F_X01/system.img --partition system_b:readonly:0:qti_dynamic_partitions_b --partition vendor_a:readonly:545050624:qti_dynamic_partitions_a --image vendor_a=out/target/product/HU_8155F_X01/vendor.img --partition vendor_b:readonly:0:qti_dynamic_partitions_b --sparse --output out/target/product/HU_8155F_X01/super.img"
2021-03-22 07:18:14 - common.py - INFO    : lpmake I 03-22 07:18:02  8264  8264 builder.cpp:937] [liblp]Partition system_a will resize from 0 bytes to 1537728512 bytes
lpmake I 03-22 07:18:02  8264  8264 builder.cpp:937] [liblp]Partition vendor_a will resize from 0 bytes to 545050624 bytes
2021-03-22 07:18:14 - build_super_image.py - INFO    : Done writing image out/target/product/HU_8155F_X01/super.img
```

### 8、lpunpack解析super.img

ref：`https://blog.csdn.net/Android_2016/article/details/103862493`

- 1、原始super.img镜像转换为ext4格式

```bash
# simg2img super.img super_ext4.img
```

- 2、提取super_ext4.img中的动态分区镜像

```bash
# xxx是输出路径，自己创建
# lpunpack super_ext4.img xxx/
#     ---> system_a.img  system_b.img  vendor_a.img  vendor_b.img
```