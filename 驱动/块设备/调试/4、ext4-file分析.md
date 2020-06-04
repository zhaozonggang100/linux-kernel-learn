### 1、ext4 layout
![ext4-layout ](./ext4-disk-layout.jpg)

### 2、分析工具

`dumpe2fs`：查看所有的块组占用的起始块和结束块以及整体的情况

`tune2fs`：和dume2fs输出的头部差不多

`e2fsck`：查看分区是否有坏块

`hexdump`：从指定位置开始取固定字节大小的块

