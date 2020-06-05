https://xgwang0.github.io/2018/12/24/Linux-FileSystem_File-Read_Write-Process/
https://xgwang0.github.io/2018/12/25/Linux-FileSystem_PageCache-VS-BlockSystem_PageCache/

### 1、page  cache

基于文件系统的读写，每个文件在内存中对应一个inode结构，可能有多个file结构，inode结构是与文件系统绑定的，inode中的address_space成员就是文件的page cache，对文件的读写都会先在page cache对应的redix树中先判断是否命令cache。

page cache以页为单位，一般是4KB

page  cache中也包含了buffer cache，为了方便文件的page cache同步到gendisk的buffer cache，减少转换

### 2、buffer cache

当使用dd命令直接操作设备文件（如/dev/mmcblkx），是直接绕过文件系统以及page cache的，使用write添加O_DIRECT也可以直接对接设备文件，但是裸IO操作必须按块对齐，一般dd默认的bs（block size是512字节），裸io直接生成bio合并到自己的plug中的request中。

这里进行读写的时候会先在设备文件的address_space的redix中判断是否缓存命令（buffer cache），命中的话直接返回，miss的bio请求对应的request就会产生一次磁盘读写。

设备文件本身处理虚拟文件系统bdevfs中，在bdevfs中每个块设备文件对应一个inode，inode里包含了块设备文件的buffer cache。

一个buffer对应块设备的一个块大小，一般是512字节

buffer被块设备inode的address_space中的page管理，一个page对应多个buffer

元数据对应的page都存放在设备文件inode结构的address_space

### 3、page cache和buffer cache的混淆

```
1、首先基于文件系统的话，文件对应的inode结构中的address_space就是文件的页缓存，这里页缓存会被分割成buffer（以块为单位），当需要同步文件page cache内容到块设备层（gendisk）时，直将将设备文件的inode中的buffer指向文件系统中的page cache管理的buffer，在这个概念上，buffer和cache是被包含关系。free命令看到的cache是指page cache，而buffer没有映射到gendisk的话，buffer列应该是不算在buffer中，如果发生了映射关系就属于buffer列了。
2、但是直接裸io操作的时候，块设备文件的inode的address_space中分配对应的buffer内存，但是Linux是以页为单位管理内存，所以buffer还是会放在page中，但这个buffer是不关联文件系统层的page cache的，所以对应的free命令，这个buffer就只对应buffer列
```

### 4、direct io

不经过文件系统的page cache，直接将bio请求提交给gendisk的请求处理函数，所以direct IO是要求扇区对齐的

场景：数据库有自己的应用层cache

### 5、bio

表示要把磁盘中的哪些block读取到内存中的哪些page，比如user读取的文件的内容大小分布在多个磁盘的block中，那么连续的block会生成一个bio请求

一个bio对应多个连续的block

### 6、plug

bio是由进程在内核态进入gendisk前生成的，但是进程不会把bio直接交给gendisk，而是有自己的pulg（槽），

进程会把bio请求合并成request放入plug中，在合适的时机unplug自己plug中的request到IO调度器的request队列中，所以进程多次产生的bio请求可能在plug中被合并成到一个request请求中。

如果进程认为自己的plug长度过大，就会unplug到IO调度器

### 7、io调度

负责将进程unplug下来的request通过合适的调度算法通过块设备驱动下发到flash设备

```
调度算法：
NOOP：简单的将相邻的request中的bio合并下发，不适合机械硬盘
DEADLINE：会对request进行排序，保证读优先，但是不会饿死写
CFQ：每个进程的bio分为实时io和非实时io
	非实时IO可以设置bio的ionice值（0~7），值越小，优先级越高
	ionice命令可以设置进程产生的io的优先级
```

### 8、dispatch队列

经过IO调度器的request终于到达块设备驱动的dispatch中了，dispatch就是块设备的request队列，块设备驱动根据设备接口的不同，将request转换成对应接口的请求命令，进行数据传输。

比如usb设备，就会通过usb host将请求下发给usb  device，并将结果返回给usb驱动，再逐级返回给应用。

如果是EMMC设备，会通过mmc host将读写请求通过mmc总线下发给EMMC

### 9、IO追踪

- ftrace

```
基于函数级别的IO追踪，可以查看IO在每一步的处理时长，判断IO慢的原因
```

- blktrace/blkparse

```
基于block的IO追踪，可以看到做了哪些IO操作
```

- iotop

```
查看进程的io流量
```

- iostat

```
每个磁盘上io的使用情况
```

- cgroup  

```
按组设置进程的IO优先级
使用cfq调度算法才有效
```