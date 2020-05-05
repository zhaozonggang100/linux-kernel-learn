### 1、概念总结  
```
1、一条usb总线上最多能接128个usb设备，由usb_bus中的devmap决定
	一个接口最多15个端点
2、USB设备的状态
	1、attached
		设备已经连接到usb接口上，是hub检测到设备的初始状态，但设备还没上电
	2、powered
		这里的上电是指设备连接到usb接口之后的状态，即使设备本身是自供电（非vbus供电），设备被复位(Reset)，或者说处于地址、配置状态。
	3、default
		在powered状态之后设备只有收到reset信号复位之后才能使用默认地址回应主机发过来的设备和配置描述符
	4、address
		表示主机分配了一个唯一的地址给设备，此时设备才能使用默认管道响应主机请求
	5、configured
		设备已经被主机配置过了，只有配置之后主机才能使用设备的所有功能
	6、suspended
		挂起状态，为了节约功耗，设备在指定的时间（3ms）内没有发生总线传输或者系统休眠，就要进入挂起状态，此时设备需要自己保存地址、配置等信息
3、USB的传输速度
	低速（1.0）：1.5Mb/s
	全速（1.0）：12Mb/s
	高速（2.0）：480Mb/s
	超速（3.0）：5.0Gb/s
4、usb packet种类（大类，大类中包括子类）
	token
	data
	handshake
	special
5、packet组成
	sync：每一帧数据都是从sync信号开始，8位二进制
	pid：区分不同的包类型，8位二进制，前四位类型，后四位校验
	address：7位总线上连接的设备或接口的地址，4位表示端点号，11位
	framenum：11位，但不是所有帧都有
	data：0~1024字节
	CRC：校验
6、四类描述符
	设备描述符：usb_device_descriptor，USB_DT_DEVICE
	配置描述符：usb_config_descriptor，USB_DT_CONFIG | USB_DT_OTHER_SPEED_CONFIG
	接口描述符：USB_DT_INTERFACE
	端点描述符：USB_DT_ENDPOINT，端点0没有专门的描述符，放在设备描述符里了
```

### 2、几个重要部分  
> 1、主机控制器  

> 2、设备  

```
数据结构：struct usb_device
1、一个设备可以是功能单一的设备或者复合功能的设备
2、一个设备可以包含多个配置，一个配置可以包含多个接口，接口对应功能
```

> 3、接口  

```
数据结构：struct usb_interface
1、一个接口代表usb设备中一个功能
2、一个接口可以包含多个端点
3、接口整体作为一个usb设备存在
4、每一个接口都需要一个独立的驱动和它交流
```

> 4 、端点  

```
端点0：
	使用message管道，可以IN、OUT
其他（最多15个）
	只能IN或者PUT
```

> 5、配置  

```
数据结构：struct usb_host_config
1、一个配置可以包含多个接口
2、一个设备同时只能属于某一个接口，接口可以通过修改配置变化
```
