参考:  

https://bean-li.github.io/EXT4-packet-meta-blocks/  
https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout  

### 1、ext4  Flexible块组

https://www.cnblogs.com/alantu2018/p/8461272.html

原理：将几个连续的块组组成一个flex_bg块组

```
作用：
	(1)  聚集元数据，加速元数据载入；
	(2)  使得大文件在磁盘上尽量连续；
```

### 2、sparse_super特性

如果超级块设置了sparse_super特性，那么超级块和块组描述表的冗余备份仅存放在编号为0,1,3,5,7的幂次方的块组中，如果未设置sparse_super特性，冗余备份存在于所有的块组中

### 3、块组

ext4文件系统被切分为多个块组。为了减少由于碎片化导致的性能下降，块分配器尽量让每个文件的块都在同一个块组内，这样可以减少seek的定位时间。块组的大小由sb.s_blocks_per_group决定，单位是块，默认情况下是8*block_size_in_bytes。在默认块大小为4KB时，每个块组会包含32768个块，总大小128M。块组的个数由设备总大小除以每个块组的大小。

所有ext4区域都是用小端存储方式。所有jbd2(日志) 都是用大端存储模式。