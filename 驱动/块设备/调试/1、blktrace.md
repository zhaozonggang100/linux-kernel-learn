参考：
http://linuxperf.com/?p=161

> 原理：

用户层使用blktrace通过debugfs将请求发送给ftrace的tracer（function、blk等），当用户层执行blktrace命令时会在/sys/kernel/debug/block/xxx/中生成dropped、msg、trace0、...、tracen（n是cpu个数-1）

`ftrace详解`：https://www.cnblogs.com/arnoldlu/p/7211249.html

`blktrace使用`：https://www.cnblogs.com/arnoldlu/p/8677812.html

### 1、aarch64使用blktrace分析IO

1、交叉编译blktrace、blkparse

2、判断系统是否挂载了debugfs

```
mount  | grep debugfs
默认debugfs会挂载到/sys/kernel/debug
```

3、首先判断当前系统支持的tracer（ftrace框架）

```
cat  /sys/kernel/debug/tracing/available_tracers，看是否支持blk

依赖文件：kernel/trace/blktrace.c，需要CONFIG_BLK_DEV_IO_TRACE支持
```

4、将blk设置为当前tracer

```
echo   blk > /sys/kernel/debug/tracing/current_tracer
```

5、打开trace

```
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo 0 > /sys/kernel/debug/tracing/events/enable		
echo 10000 > /sys/kernel/debug/tracing/buffer_size_kb    //每cpu缓存大小
```

6、ftrace其他配置

```
/sys/kernel/debug/tracing/
	buffer_size_kb：单个CPU动态跟踪信息缓存大小
	buffer_total_size_kb：所有cpu动态跟踪信息缓存大小
```

