### 1、概要

### 2、使用

- 1、确认debugfs的挂载路径

```
debugfs一般会挂载到/sys/kernel/debug目录下，该目录下tracing目录包含用来配置当前使用的trace的文件
/sys/kernel/debug/tracing/current_tracer
```

- 2、选择要使用的tracer

```
1、查看当前系统支持的tracer
	cat /sys/kernel/debug/tracing/available_tracers
	blk function_graph preemptirqsoff preemptoff irqsoff function nop
2、从上面的列表中选择要使用的tracer
	如：echo function > /sys/kernel/debug/tracing/current_tracer
```

- 3、使能ftrace

```
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

- 4、查看trace

```
cat /sys/kernel/debug/tracing/trace
```

### 3、各种tracer的使用

首先在/sys/kernel/debug/tracing/目录下有README，介绍ftrace的使用

- 1、function

```
功能：跟踪函数调用，默认会跟踪所有函数的调用
指定要跟踪的函数：echo function_name > /sys/kernel/debug/tracing/set_ftrace_filter
取消某个函数的跟踪：echo '!function_name' > /sys/kernel/debug/tracing/set_ftrace_filter 或者 echo function_name > /sys/kernel/debug/tracing/set_ftrace_notrace
```

- 2、function_graph

```
功能：相比function能知道更详细的函数调用上下文
跟踪某个函数的调用堆栈：echo function_name > /sys/kernel/debug/tracing/set_graph_function
```

- 3、event

```
功能：监听系统事件
打开调度事件监听：
	echo nop > /sys/kernel/debug/tracing/current_tracer
	echo 1 > /sys/kernel/debug/tracing/events/sched/enable
```

