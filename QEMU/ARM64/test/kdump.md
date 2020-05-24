操作流程：<https://blog.csdn.net/lsy673908720/article/details/90215501>

kexec原理：<https://zhuanlan.zhihu.com/p/105284305>

kdump原理：<https://www.jianshu.com/p/1645da328c5e>

dump  file：/proc/vmcore



### 1、crash调试

arm64编译crash：<https://www.jianshu.com/p/7d998f474033>

github：https://github.com/crash-utility/crash.git

crash   vmcore  -d 12 vmlinux：开启crash  debug

crash   vmcore  vmlinux --machdep vabits_actual=48：将虚拟地址设置为48，不然会标错虚拟地址不匹配（<https://github.com/crash-utility/crash/issues/52>）

crash命令

<https://blog.csdn.net/yuxinghai2008/article/details/36181193>

<https://www.jianshu.com/p/ad03152a0a53>

```
bt：打印call  trace
ps：打印系统奔溃时在运行的进程
set pid：连接debug系统奔溃时正在运行的进程，上下文切换
dmesg：查看系统奔溃时dmesg信息
dis  start_kernel:汇编指定函数
bt -slf：显示出错函数所在的文件，函数偏移
dis -r <地址>：反汇编bt -slf出来的函数地址
bt -t：如果栈呗破坏了，导出整个堆栈
bt -c <cpunum>：查看指定cpu的堆栈
log：和dmesg类似
结构体名：查看结构体数据成员
rd -a linux_banner：查看系统版本信息
rd 内存地址 大小：查看内存内容
rd 内存地址 大小 -S：打印内存地址开始处指定大小的符号表
ps -t pid：查看线程运行时间
ps -A：查看活动线程
ps -k：查看内核线程
ps -u：查看用户线程
ps -r pid：查看线程资源
set -p：切到panic线程
module：查看当前加载的符号表
file：查看打开的文件
irq：查看irq信息
kmem：查看kernel内存
mach：查看平台信息
mount：查看挂载的文件系统
```



### 2、gdb调试



