### 1、概述

fio是测试磁盘性能的比较好用的工具，主要测试iops

下载源码地址：https://brick.kernel.dk/snaps/

https://fio.readthedocs.io/en/latest/fio_doc.html

### 2、aarch64源码编译

- 1、./configure --cpu=aarch64 --cc=aarch64-linux-gnu-gcc
- 2、修改Makefile文件，CFLAGS+=--static
- 3、make生fio二进制

### 3、测试场景

- 1、基于文件系统
- 2、基于裸设备

