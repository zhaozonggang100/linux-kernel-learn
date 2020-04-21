### 1、uboot加载Linux镜像  

> uboot只能加载uimage格式的镜像（有一个64字节的头）  
	
1、从NAND flash中加载  
```
1、获取环境变量bootargs（传递给内核的参数）
2、从nandflash的mtd分区表中解析kernel分区的起始地址和大小
3、获取内核镜像的头64字节，确定将内核加载到内存中的地址，通过nand命令将镜像加载到内存
4、bootm 0x xxxx xxxx会激活do_bootm函数
5、将内核重定位到内核运行地址
6、给内核传递参数
7、启动内核
8、跳转到解压缩程序
```
2、从emmc加载内核  
```
从emmc加载镜像uimage到内存的命令为mmc
```

### 2、ARM架构Linux内核初始化过程  

1、uimage依赖关系：
```
build/vmlinux		
//去除调试信息
	--->build/arch/arm/boot/Image		
	//根据build/arch/arm/kernel/vmlinux.lds将压缩内核、启动代码、解压代码生成compressed/vmlinux
		--->build/arch/arm/boot/compressed/vmlinux
		//通过objcopy生成zImage
			--->build/arch/arm/boot/zImage
			//加上64字节的头生成uImage
				--->build/arch/arm/boot/uImage(用来uboot加载)
	uimage是通过uboot提供的mkimage工具编译出来的，提供了内核加载到内存中地址、大小、内核的入口函数、镜像类型
```
	
2、build/vmlinux是根据arch/arm/kernel/vmlinux.ld.S链接脚本生成，脚本中指定内核运行时起始虚拟地址为PAGE_OFFSET + TEXT_OFFSET = 0xC000 0000 + 0x0000 8000  
	
3、压缩后的vmlinux的.start段中包含内核的解压代码  
	
4、初始化过程  
```
1、bootloader加载内核镜像到内存中，压缩内核镜像通过调用decompress_kernel（在zImage的头部）将压缩内核解压到主存中，跳转到内存中解压后image的内存地址执行第二阶段（汇编实现：/arch/arm/kernel/head.S中）
uboot会为加载Linux初始化内存等硬件，为Linux提供初始环境
此时内核镜像在主存的1MB开始地址处，当后面会将该1M地址处+PAGE_OFFSET作为内核镜像（代码段+数据段）的起始虚拟地址，起始虚拟地址是0xC010 0000，0~1M的物理地址用于Linux启动初始阶段存放初级页表使用
		
2、stext是解压后内核汇编入口函数
	1、设置处理进入SVC模式，关闭中断
	2、获取处理器类型从cp15寄存器中并判断当前内核机器代码是否支持该处理器
	3、创建页表映射主存前1M的物理地址到Linux虚拟地址0xC000 0000 ~ 0xC010 0000(为了kernel的运行)这1M的空间包含了内核的初始页表等
	4、开启MMU进入第三阶段（c语言start_kernel）
		
3、c语言入口start_kernel，此时内核访问的是虚拟地址	
	1、setup_arch() 进行与硬件架构相关的初始化	//	arch/arm/kernel/setup.c
		1、处理器初始化
		2、根据uboot传递的参数，从内存中加载设备树信息到设备描述结构中
		3、paging_init创建页表映射所有的物理内存、IO空间free_area_init会初始化mem_map[]数组，这个数组包含了所有的物理页
	2、初始化异常向量表和中断处理函数
	3、初始化调度器和时钟中断
	4、console初始化
	5、cache初始化
		动态内存分配
		VFS
		页缓存
	6、初始化内存管理pagecache_init
	7、IPC初始化，最后调用arch_call_rest_init进行剩余的初始化
		init进程创建
		初始化内核模块
		挂载根文件系统
		网络协议栈初始化（inet_init）
		...
		启动init进程
```