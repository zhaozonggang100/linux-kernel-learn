参考：

http://www.wowotech.net/memory_management/zram.html

### 1、系统配置

- 1、zram使用的压缩算法

` /sys/block/zram0/comp_algorithm `

- 2、zram的大小

`  /sys/block/zram0/disksize `

- 3、使能zram

` mkswap /dev/block/zram0` ---> `swapon /dev/block/zram0`

- 4、失能zram

`swapoff /dev/block/zram0`

- 5、压缩率

` /sys/block/zram/mm_stat `

6、查看换出换入总量

` /proc/vmstat`

7、zram每次压缩的页数量

` /proc/sys/vm/page-cluster `

