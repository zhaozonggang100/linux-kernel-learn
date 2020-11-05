### 1、概要

 Perf 是内置于Linux 内核源码树中的性能剖析（profiling）工具。它基于事件采样原理，以性能事件为基础，支持针对处理器相关性能指标与操作系统相关性能指标的性能剖析。可用于性能瓶颈的查找与热点代码的定位。linux2.6及后续版本都自带该工具，几乎能够处理所有与性能相关的事件。 

### 2、安装

- 1、debian系统安装

方法1：直接apt安装

`apt install linux-perf`

方法2：先安装Linux源码到/usr/src中，源码中包含了perf工具，进程编译安装

```
1、sudo apt install linux-source
2、cd /usr/src/linux目录/tools/perf
3、sudo make
4、sudo make install
```

### 3、使用场景

- 1、分析进程为什么cpu占用率高

```
1、首先top找到占用cpu高的进程
2、perf top -g -p 进程pid
	-g开启调用关系分析
	-p指定占用率高的进程的进程号
	
	可以显示出哪个函数占用cpu高，从而分析该函数到底在执行什么操作
```



