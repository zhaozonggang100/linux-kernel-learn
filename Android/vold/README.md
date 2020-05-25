```
https://blog.csdn.net/dengziliang001/article/details/51790559（vold实现总览）
https://blog.csdn.net/dengziliang001/article/details/51776383（vold->kernel）
```



### 1、概述

处于framework和Linux之间，通过netlink和Linux交互（接受usb事件等），通过unix域套接字和

### 2、file

/dev/socket/vold：unix域套接字和framework层交互

/dev/block/vold

/etc/vold.fstab

