### 1、介绍  

### 2、使用场景  
1、iptables、ebtables、arptables  
2、iproute2  
```
工具：
ip
tc
ss
lnstat
bridge
ss
内核接口
rtnetlink_net_init初始化用于接收用户空间数据的回调函数
```
	
3、net-tool（基于IOCTL）  
```
ifconfig
arp
route
netstat
hostname
rarp
```

4、seLinux  

5、audit（内核审计）  
	
6、uevent  
```
功能：内核空间向用户空间发送数据
内核接口：uevent_net_init
```
	
7、无线  
```
iw
hostapd
```

8、udev、mdev  
```
字符设备热插拔
```

### 3、使用  
- 1、用户层  
```
直接使用socket接口，指定AF_NETLINK协议族
```

- 2、内核层  
```
struct sock * netlink_kernel_create(struct net *net, int unit, struct netlink_kernel_cfg *cfg)
	net：网络命令空间
	unit：使用的netlink消息类型，最多有32种，用户态和内核态通过相同的unit标识建立连接
```