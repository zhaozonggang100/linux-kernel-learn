### 1、寄存器

https://blog.csdn.net/weixin_42625444/article/details/90636701

`CID`：卡身份识别寄存器 128bit,只读， 厂家号，产品号，串号，生产日期。

`RCA`： 卡地址寄存器，可写的16bit寄存器，存有Device identification模式由host分配的通信地址，host会在代码里面记录这个地址，MMC则存入RCA寄存器，默认值为0x0001。保留0x0000以用来将all device设置为等待CMD7命令状态。

`CSD`: 卡专有数据寄存器部分可读写128bit，卡容量，最大传输速率，读写操作的最大电流、电压，读写擦出块的最大长度等。

`SCR`: 卡配置寄存器， 可写的 64bit 是否用Security特性(LINUX不支持)，以及数据位宽（1bit或4bit）。

`OCR`: 卡操作电压寄存器 32位， 只读，每隔0.1V占1位， 第31位卡上电过程是否完成。

### 2、卡的识别过程

http://bbs.eeworld.com.cn/thread-1060619-1-1.html

MMC通过发CMD的方式来实现卡的初始化和数据访问

```
1、CMD1从idle状态进入Ready State状态（失败进入Inactive State状态），获取设备的OCR参数完成EMMC的初始化，包括memory寻址方式，和host电压是否匹配等
2、CMD2从Ready State状态进入Identification State，获取设备的CID完成设备的信息配置
3、CMD3从Identification State进入Data Transfer Mode状态，host发送16bit的RCA设置EMMC相对host的通信地址
```

