参考：

https://blog.csdn.net/feelabclihu/article/details/105502169

https://blog.csdn.net/shenjin_s/article/details/79761425?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-10.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-10.nonecase



### 1、概述

UFS是目前广泛应用于移动设备的闪存，传输速度默认1248MB/S，可以配置为4096MB/S

### 2、架构

ufs采用四层结构设计，从上到下依次为

- a. 应用层

```
包含了任务管理（task manager）、ufs command set layer（ucs）子层
1、任务管理
	处理command queue来控制任务，比如abort
2、ucs
	处理command（比如读写），主要将scsi command转换为原生的ufs command
3、设备管理（device manager）
	处理一些设备电源功耗相关的command，比如suspend、resume、power down、别的一些电源操作
	可以通过两个服务控制host controller：
		UDM-SAP：比如通过UTP获取设备描述符
		UIO-SAP：比如通过UIC（位于传输层和数据链路层之间作为抽象接口为上层提供服务）对host进行reset
```



- b. 传输层（UTP）

```
1、传输层类似tcp/ip的传输层，host端的传输层对应device端的传输层，将应用层（device manager和application）的命令转换为UFS protocol information units (UPIU) 
2、传输层为上层提供了三个服务
	a、UDM_SAP – Communicates with the device manager
		处理dm的设备管理命令
	b、UTP_CMD_SAP – Transports commands
		处理ucs生成的ufs command（比如read/write）
	c、UTP_TM_SAP – Transports task management functions
		处理task management生成的command（比如设备abort等）
3、传输层使用CS模型
	主机端相当于client，device的LUN（logiccal unit）相当于server
	device的么一个LUN都有一个task queue，同一时间只能服务一个task，不同的LUN可以同时服务于一个task
4、一个典型的命令时序
	COMMAND UPIU
	DATA out upiu （可选）
	DATA in upiu（可选）
	Response UPIU
	最终的数据传输是通过DMA
```



- uic（ufs interconnter layer）

1、为传输层和应用层提供服务接口，包括两个服务

​	UIC_SAP – Transports UPIU from the UFS host to the UFS device

​			服务于传输层

​	UIO_SAP – Transports queries and control of device management	

​			服务于设备管理层

2、该层主要包含了UFS四层中的下面两层

c. 数据链路层（MIPI UniPro）
d. 物理层（MIPI M-PHY）



### 3、物理连接

ufs属于一种通用闪存规范，通信的两端包括host controller（一般是soc内部集成）和device（闪存设备），两端通过时钟线、复位线、控制输出、传输输出、控制输入、传输输入

- host

  host的控制器通过MIPI规范的UIC物理接口和device端的UIC物理接口连接

  两端的UIC通过上面描述的6根物理线连接

### 4、LUN

LUN是位于闪存设备的独立单元，用来处理host传输过来的command，一个UFS设备最多能有8个LUN