## 1、概要

http://blog.nsfocus.net/bpftrace-dynamic-tracing-0828/

 由于bpftrace是建立在eBPF之上的一种编程语言，考虑到部分特性需满足Linux内核的支持，因此建议Linux的内核版本在4.9以上 

## 2、安装

- debian系统

```
1、sudo apt-get install -y libbpfcc-dev
2、sudo apt-get install -y bison cmake flex g++ git libelf-dev zlib1g-dev libfl-dev systemtap-sdt-dev binutils-dev
3、sudo apt-get install -y llvm-7-dev llvm-7-runtime libclang-7-dev clang-7
4、git clone https://github.com/iovisor/bpftrace
5、mkdir bpftrace/build; cd bpftrace/build;
6、cmake -DCMAKE_BUILD_TYPE=Release ..
7、make -j8
8、sudo make install


当然也可以apt安装
sudo apt install -y bpftrace

或者docker镜像下载
docker run -ti -v /usr/src:/usr/src:ro \
       -v /lib/modules/:/lib/modules:ro \
       -v /sys/kernel/debug/:/sys/kernel/debug:rw \
       --net=host --pid=host --privileged \
       quay.io/iovisor/bpftrace:latest \
       capable.bt
```

### 3、运行

机制：运行.bt文件（./xxx.bt或者bpftrace xxx.bt 参数1 参数2 ...）或者直接命令行执行bpftrace -e program，程序会一直检测程序中定义的事件，直到手动停止（CTRL+C or exit()）

- 1、工具路径

```
1、The bpftrace binary will be in installed in /usr/local/bin/bpftrace
2、tools in /usr/local/share/bpftrace/tools

备注：You can change the install location using an argument to cmake, where the default is -DCMAKE_INSTALL_PREFIX=/usr/local
```

- 2、简单案例  

1、列出当前版本内核所支持Kprobes探针列表
`bpftrace -l 'kprobe:tcp*'`

2、 列出当前版本内核所支持的Tracepoints探针列表 

` bpftrace -l 'tracepoint:*' `

3、打印hello

` bpftrace -e 'BEGIN { printf("hello world\n"); }' `

4、追踪文件打开

` bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }' `

### 4、程序结构

- 1、.bt文件头部声明解析器

`#!/usr/bin/env bpftrace`



- 2、bpftrace程序是一系列具有相关操作的探针（probes），当probes执行相关的操作之前可以包含一个可选的过滤器表达式，只有在过滤器表达式为true时，才会触发该操作，如以下：

```bpftrace
# 不带过滤表达式的探针操作
probes { actions }
# 带过滤表达式的探针操作
probes /filter/ { actions }
```



- 3、probe

探针以探针类型名称开头，然后以冒号分隔的层次结构

`type:identifier1[:identifier2[...]]`



层次结构由探针类型定义，例如：

```
1、kprobe探针类型检测内核函数调用，只需要一个标识符:内核函数名
kprobe:vfs_read

2、uprobe探针类型用于检测用户级别的函数调用，并且需要binary路径和函数名称
uprobe:/bin/bash:readline
```



可以使用逗号分隔符指定多个探针以执行相同的操作

`probe1,probe2,...{actions}`



有两种不需要额外标识符的特殊探针类型：bpftrace程序的开头和结尾均使用BEGIN和END触发



- 4、过滤器

格式：/条件/

例子：

​		/pid == 123/

​		/pid != 123/

​		/pid/

​		/pid > 100 && pid < 1000/



- 5、action

action可以是单个语句，也可以是由分号分隔的语句

`{action one; action two ; ...}`



### 4、bpftrace变量

- 1、built-in

| 变量名 | 含义 |
| :-----:| :----: |
| pid | 进程id |
| tid | 线程id |
| uid | 用户id |
| nsecs | 纳秒级时间戳 |
| cpu | 处理器id |
| comm | 进程名 |
| retval | 返回值 |
| func | trace函数名 |
| probe | probe全名 |
| kstack | 多行字符串的形式返回内核级堆栈 |
| ustack | 多行字符串形式返回用户级堆栈 |
| args | 参数 |

位置参数：$1,$2...

- 2、scratch

临时变量，具有$前缀，他们的名字和类型是在第一次分配时设定的，只能在分配了他们的操作块中使用

```
# 将$x声明为整数
$x = 1;
# 将$y声明为字符串
$y = "hello";
# 将$z声明为指向结构体task_struct的指针
$z = (struct task_struct *)curtask
```

- 3、map

映射变量使用BPF map存储对象，并具有@前缀，它们可用于全局存储，在操作之间传递数据

格式如下：

```
@name
@name[key]
@name[key1,key2[,...]]
```



### 5、函数

> 1、内置函数

- 1、printf()

使用方法和c语言一样

- 2、join(char *arr[])

将多个带有空格字符串的字符串数组连接起来并打印出来

- 3、str(char *)

从指针返回字符串

- 4、kstack(mode[,limit])

返回内核级堆栈

- 5、ustack

返回用户级堆栈

- 6、ksym()

将地址解析为其符号名称

- 7、usym()

将地址解析为其名称

- 8、system()

在shell上运行命令



> 2、map函数

