参考：
https://blog.csdn.net/android_2016/article/details/102912469

https://www.freesion.com/article/4354248615/（差分包写入步骤）

### 1、概述  

使用A/B方式进行OTA，替换recovery升级方式

### 2、源码路径  

```bash
system/update_engine
```

### 3、解析payload.bin中的operations

```bash
# payload.bin在解压后的OTA包内
system/update_engine/scripts/payload_info.py --list_ops payload.bin
```