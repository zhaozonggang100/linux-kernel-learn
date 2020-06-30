---
参考：
https://www.jianshu.com/p/dd8ab6b68c6a
https://www.zhihu.com/question/325947168
---



### 1、概述

两大功能：

- 外设DMA地址映射到物理地址
- 中断重映射

优点：

- 当没有IOMMU的时候，外设可以通过DMA访问到所有的物理地址空间

特点：

- iommu是将设备访问的虚拟地址转换为物理地址
- mmu是将cpu访问的虚拟地址转换为物理地址

