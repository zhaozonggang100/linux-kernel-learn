### 1、内存初始化概述  
```
内核启动的时候如何知道物理内存的大小以及如何初始化内存，DTS中的memory字
段描述了内存的起始地址（cpu物理寻址空间）和大小，内核解析设备树通过memblock_add将内存信息添加到memblock子系统中  
```

### 2、初始化步骤  
参考：https://www.cnblogs.com/alantu2018/p/8447576.html  

- 1、使用内存之前需要先初始化页表  

- 2、初始化ZONE  

- 3、buddy系统初始化  

- 4、slab分配器初始化  

