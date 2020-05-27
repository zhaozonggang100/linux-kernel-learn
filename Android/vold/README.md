```
https://blog.csdn.net/dengziliang001/article/details/51790559（vold实现总览）
https://blog.csdn.net/dengziliang001/article/details/51776383（vold->kernel）
https://blog.csdn.net/frank_zyp/article/details/56667175（VoldConnect线程）
https://blog.csdn.net/u010142953/article/details/80425407（usb热插拔）
```



### 1、概述

处于framework和Linux之间，通过netlink和Linux交互（接受usb事件等），通过unix域套接字和framework（MountService创建的VoldConnect线程）交互

### 2、system file

/dev/socket/vold：unix域套接字vold和framework（native）层交互

/dev/block/vold

/etc/vold.fstab

### 3、source

xxxx/android/system/vold

