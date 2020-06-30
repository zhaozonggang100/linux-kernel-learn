参考：

https://blog.csdn.net/armwind/article/details/53454251



### 1、概述

ION是google为了防止内存碎片引入的内存管理器

### 2、内核驱动

```
drivers/staging/android/ion/

提供了ioctl函数给用户空间

设备树：arch/arm64/boot/dts/qcom/<target>-ion.dtsi
```

### 3、用户空间

```
用户空间可以使用libion，也可以直接使用ion  ioctl接口
```

