### 1、介绍  

### 2、netfilter子系统提供功能  
- 1、数据包过滤  
- 2、数据包选择（iptables）  
- 3、网络地址转换（NAT）  
- 4、数据包操纵（在路由选择之前和之后修改数据包报头的内容）  
- 5、连接跟踪  
- 6、网络统计信息收集  

### 3、netfilter挂载点  
- 1、NF_INET_PRE_ROUTING  
```
ip_rcv
ipv6_rcv
```

- 2、NF_INET_LOCAL_IN  
```
ip_local_deliver
ip6_input
```
3、NF_INET_FORWARD  
```
ip_forward
ip6_forward
```
4、NF_INET_LOCAL_OUT  
```
__ip_local_output
__ip6_local_output
```
5、NF_INET_POST_ROUTING  
```
ip_output
ip6_finish_output2
```

-  给挂载点添加钩子函数  
```
static inline int NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk, struct sk_buff *skb,struct net_device *in, struct net_device *out,int (*okfn)(struct net *, struct sock *, struct sk_buff *)) {
	int ret = nf_hook(pf, hook, net, sk, skb, in, out, okfn);            
	if (ret == 1)
		ret = okfn(net, sk, skb);
	return ret;
}
返回值：
	NF_DROP：丢弃数据包
	NF_ACCEPT：由网络协议栈继续处理
	NF_STOLEN：数据包不继续传输，由钩子函数进行处理
	NF_QUEUE：将数据包排序，由用户空间处理
	NF_REPEAT：再次调用钩子函数
```

### 4、netfilter的表 
- 1、FILTER表  

1、作用  
多用于本地和转发过程中数据过滤  

2、包含的链  
```
INPUT
OUTPUT
FORWARD
```

- 2、NAT表  

1、作用  
多用于源地址/端口转换和目标地址/端口的转换  

2、包含的链  
```
PREROUTING
OUTPUT
POSTROUTING
```

- 3、MANGLE表  

1、作用  
可实现拆解报文、修改报文、重新封装，可常见于IPVS的PPC下多端口会话保持  

2、包含的链  
```
PREROUTING
INPUT
FORWARD
OUTPUT
POSTROUTING
```

- 4、raw表  

1、作用  
决定数据包是否被状态跟踪机制处理，需关闭nat表上的连接追踪机制  

2、包含的链  
```
PREROUTING
OUTPUT
```