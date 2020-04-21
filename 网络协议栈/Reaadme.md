### 1、网络协议栈概述  
网络协议栈作为Linux的重要组成部分，实现了万物互联，整个协议栈可以简单的分为三大部分  

```
1、socket层
2、传输层、网络层（L4、L3）
3、数据链路层（L2）
```
### 2、网络协议栈的初始化  

- 协议栈的初始化主要分为两大部分

> 1、网卡驱动（L1：物理路层，该部分不应该归类于网络协议栈）  

网卡驱动的初始化需要根据编译Linux时选择的不同型号的网卡设备进行初始化，属于设备驱动部分  

> 2、内核通用层（L2、L3、L4、SOCKET层）  

网络协议栈其他部分属于内核的组件，在内核初始化时遍历所有的init段中的函数指针执行相应模块的初始化，网路协议栈部分不能被编译成module  

- 主要协议的初始化  
```
1、subsys_initcall(net_dev_init)     //net/core/dev.c，L2
	初始化对应的net_device结构体，每个网络设备在内核中都会对应一个net_device结构
2、fs_initcall(inet_init)            //net/ipv4/af_inet.c，L3、L4
	ipv4部分初始化，这里会初始化网络层（ip），传输层tcp、udp、icmp等
3、subsys_initcall(neigh_init)       //net/core/neighbour.c，L3
	arp协议初始化
4、pure_initcall(net_ns_init);       //net/core/net_namespace.c，L3
	network  namespace初始化
5、core_initcall(netlink_proto_init) //net/netlink/af_netlink.c
	netlink初始化
6、core_initcall(sock_init);         //net/socket.c
	socket层初始化
```

### 3、网络协议栈重要的知识点  
- 1、事件通知链  
```
1、介绍  
	网络设备状态发生变化时通知内核协议栈
2、使用
	1、创建通知链
		#define RAW_NOTIFIER_HEAD(name) \
			struct raw_notifier_head name =
				RAW_NOTIFIER_INIT(name)
		网络子系统创建了3个通知链：
			inet_addr：网络设备的ipv4地址发生变化时使用
			inet6_addr：网络设备的ipv6地址发生变化时使用
			netdev_chain：网络设备注册、状态发生变化时使用
		
	2、注册事件处理函数到指定的通知链
		notifier_chain_register(struct notifier_block**nl,struct notifier_block *n)
	3、网络子系统用到的通知链场景
		1、reboot_notifier_list
			网络子系统注册事件处理函数到该链，当系统重启时会通知网络子系统
		2、路由子系统
			路由子系统使用ip_fib_init向inet_addr和netdev_chain两个通知链上注册回调函数
```

