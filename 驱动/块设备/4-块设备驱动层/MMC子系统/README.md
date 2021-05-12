参考：

https://blog.csdn.net/ooonebook/article/details/55001322

https://www.cnblogs.com/RandyQ/p/3607107.html

https://blog.csdn.net/yuesichiu/article/details/74159253

### 1、概述

MMC子系统为Linux上层提供了访问符合MMC协议规范的MMC CARD（包括EMMC、SD卡、SDIO接口的卡：wifi、bt）的接口，对下层的不同host控制器控制的MMC CARD提供了抽象层，使得接入更加方便

### 2、源码目录

> drivers/mmc

- +host  

```
1、概述
	soc上一般会有host controller用来和mmc card中的controller通信，从而控制card
```

- +core

```
1、概述
	core用来管理host和card，抽象出三种虚拟总线
	1、platform_bus_type
		管理host driver 和 host device，采用名称匹配
	2、mmc_bus_type
		管理mmc card 和 mmc driver，永远能匹配成功
	3、sdio_bus_type
		sdio类型的device 和 driver
```

- +card

```
1、概述
	crad驱动注册到mmc_bus_type之后会自动匹配到core事先注册的card设备，从而执行probe函数，会注册注册块设备给上层（块设备IO调度器）访问
```



