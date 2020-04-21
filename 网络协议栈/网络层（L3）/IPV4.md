### 1、报文格式  
- 1、长度  
ip头占20~60字节，ip选项0~40字节  

- 2、表示数据结构  
struct iphdr  

### 2、初始化  
- 1、api  
```
1、inet_init()--->ip_init()
	ipv4网络层初始化
2、inet_init()--->dev_add_pack(&ip_packet_type)
	注册ip层接收以太网数据包的回调函数ip_rcv
```

### 3、L2-->L3层数据包流向   
> 此处数据包会先经过PREROUTING链钩子函数的过滤，接着判断数据包的流向  

- 1、发往本地的  
```
ip_rcv--->.....--->ip_local_deliver 的报文接着会通过INPUT链 
ip_rcv--->.....--->ip_local_deliver_finish  
```

- 2、转发的  
```
ip_rcv--->.....--->ip_forward处理的报文会通过FORWARD链  
```

### 4、L4--->L3层数据包处理  
- 1、tcp协议处理  
```
ip_queue_xmit：提供自己处理分段的传输层协议,最终调用ip_local_out发送数据包，此时会检测OUPUT链是否有钩子函数
ip_build_and_send_pkt:发送SYN、ACK报文
```

- 2、udp协议处理  
```
ip_append_data：提供不处理分段的传输层协议，不发送数据包，只是生成数据包
ip_finish_skb：实际发送数据包函数，调用ip_send_skb发送数据包
```

- 3、icmp协议处理  
```
ip_append_data：提供不处理分段的传输层协议，不发送数据包，只是生成数据包
ip_push_peding_frames:实际发送报文的函数，最终调用ip_send_skb发送数据包，再调用ip_local_out，原始套接字也调用该方法
```

### 5、网络层原始套接字  
参考：https://blog.csdn.net/lixiangsheng2012/article/details/83421442
在用户态组织ipv4报文头，直接调用dst_output发送给数据给ip层  

### 6、ip分段  
- 分段函数：ip_fragment  
- 重组段函数：ip_defrag  
- 调用点：
```
ip_local_deliver
	先判断是否分片，分片则调用ip_defrag重组
	接着会检测INPUT链的钩子函数，通过则调用ip_local_deliver_finish
ip_call_ra_chain
```