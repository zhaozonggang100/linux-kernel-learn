### 1、HUB工作队列何时添加工作项处理hub端口插拔事件  
> usb_wakeup_notification  

```
设备或者端口状态发生变化xhci中断处理函数调用
```

> hub_irq  

```
urb发包时
```

> hub_port_logical_disconnect  

> hub_activate  

> usb_hc_died--->usb_kick_hub_wq  

```
HCD异常断开，如果HCD是PCI总线设备则会自动调用，否则需要手动调用
```

### 2、hub驱动  
1、不管有多少个hub，hub的驱动只有一个  
2、每一个hub（接口）只有一个端点（0端点除外，0端点是必须的）及中断端点，hub的传输是中断传输  
3、控制端点只在usb设备（非hub）中有  
