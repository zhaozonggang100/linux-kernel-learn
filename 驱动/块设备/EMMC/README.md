### JEDEC
固态固态技术协会是微电子产业的领导标准机构，下载emmc spec
https://www.jedec.org/
账号：Akon
密码：default
邮箱：18600458483@163.com


### 1、EMMC特性

```
520mwx.com/view/63589
1、必须先擦再写，只能将当前为1的比特位改为0，不能将0的比特位改为1，只有擦除可以将0改为1
2、每一个块擦写次数有限（十万到百万次不等），超过擦写次数，无法提供可靠的数据存储，成为坏块
3、为了延长整个flash memory的寿命，需要在软件层面做到读写均衡
4、除了擦写导致的坏块之外，生产本身也会制造出坏块，称为固有坏块
5、读写干扰：由于硬件实现上的物理特性，Flash Memory 在进行读写操作时，有可能会导致邻近的其他比特发生位翻转，导致数据异常。这种异常可以通过重新擦除来恢复。Flash Memory 应用中通常会使用 ECC 等算法进行错误检测和数据修正。
6、电荷泄露：存储在 Flash Memory 存储单元的电荷，如果长期没有使用，会发生电荷泄漏，导致数据错误,不过这个时间比较长，一般十年左右。此种异常是非永久性的，重新擦除可以恢复。
7、nandflash的读写都是按块操作
8、EMMC中实现NANDFLASH管理（坏块管理、擦写均衡、ECC-纠错、垃圾回收等）的软件叫FTL（FLASH Translation Layer）
```

### 2、EMMC的可靠写

### 3、数据标签机制

### 4、EMMC MODE

https://linux.codingbelief.com/zh/storage/flash_memory/emmc/emmc_modes.html

- 1、boot mode

- 2、device identification mode  

	emmc在退出boot模式或者没有使能boot模式，会进入设备识别模式，等待host发送cmd1并作出响应
  
- 3、ready state mode  
	
	emmc初始化完成之后会进入ready state mode，等待host发送cmd2，获取emmc的device cid（Device identification number，它包含了 eMMC Device 的制造商、OEM、设备名称、设备序列号、生产年份等信息，每一个 eMMC Device 的 CID 都是唯一的，不会与其他的 eMMC Device 完全相同）
	eMMC Device 接收到 CMD2 后，会将 127 Bits 的 CID register 的内容通过 Response 返回给 Host。
	
- 4、device identification mode  

	这次进入设备识别模式，会等待Host 会发送参数包含 16 Bits RCA 的 CMD3 命令，为 eMMC Device 分配 RCA  

	设定完 RCA 后，eMMC Devcie 就完成了 Devcie Identification，进入 Data Transfer Mode。  
	RCA 是在 Devcie Identification 过程中，由 Host 分配的 16 Bits 的设备地址，主要用于 Data Transfer Mode 下进行通信时，选定具体要进行操作的 eMMC Devcie。  
	
	Host 分配的 RCA 通常从 1 开始递增，0 地址作为广播地址。eMMC Devcie 的 RCA register 保存了 Host 分配的 RCA。  
	
	TODO：确认掉电重启后，RCA register 的值是否丢失。  
	
- 5、data transfer mode  

	当设备识别模式结束之后会进入数据传输模式  
	eMMC Device 完成 Device Identification 后，就会进入到 Data Transfer Mode 的 Standby State。  
	
	在 Standby State 时，Host 可以通过发送 CMD5 命令，让 eMMC Devcie 进入 低功耗的 Sleep State，而后再发送 CMD5 命令则可以让 eMMC Device 退出 Sleep State。  
	
	在 Standby State 时，Host 可以通过发送 CMD7 命令，让 eMMC Devcie 进入 Transfer State，而后再发送 CMD7 命令则可以让 eMMC Device 退出 Transfer State。  

- 6、interrupt mode  

  当进入数据传输模式之后，host可以发起命令让emmc处于中断模式，当数据传输结束时通过中断reponse host  

