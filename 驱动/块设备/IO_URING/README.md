

https://zhuanlan.zhihu.com/p/62682475?utm_source=wechat_timeline

https://yq.aliyun.com/articles/707076

https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/

https://www.mdeditor.tw/pl/pAve（含有examples）

# 1、liburing

io_uring的用户态库，github链接如下

```
https://github.com/axboe/liburing
```

# 2、qemu + linux5.2 + arm64 测试io_uring性能

### 1、搭建qemu环境，参考如下链接

```
https://biscuitos.github.io/blog/Linux-5.2-arm64-Usermanual/
```

### 2、交叉编译liburing

```
1、从第一步的链接git clone源码
2、修改configure中编译器
	c=${CC:-/home/akon/src/akon/BiscuitOS/output/linux-5.2-aarch/aarch64-linux-gnu/aarch64-linux-gnu/bin/aarch64-linux-gnu-gcc}
	cxx=${CXX:-/home/akon/src/akon/BiscuitOS/output/linux-5.2-aarch/aarch64-linux-gnu/aarch64-linux-gnu/bin/aarch64-linux-gnu-g++}
3、./configure
4、make
```

### 3、测试

- 1、使用liburing库自带的examples
- 2、基于fio使用io_uring engine测试iops

[github源码链接](https://github.com/axboe/fio.git)

[fio官方解释io_uring](https://fio.readthedocs.io/en/latest/fio_doc.html)

> 编译

1、git clone fio源码

git clone https://github.com/axboe/fio.git

2、设置libaio环境

```bash
/home/akon/src/akon/BiscuitOS/output/linux-5.2-aarch/aarch64-linux-gnu/aarch64-linux-gnu/aarch64-linux-gnu/libc/usr/include/ 中添加libaio.h
/home/akon/src/akon/BiscuitOS/output/linux-5.2-aarch/aarch64-linux-gnu/aarch64-linux-gnu/aarch64-linux-gnu/libc/usr/lib 中添加liburing.a liburing.so libaio.so

修改config
diff --git a/configure b/configure
index 39a9248d..4c15245e 100755
--- a/configure
+++ b/configure
@@ -654,9 +654,9 @@ int main(void)
 }
 EOF
   if test "$libaio_uring" = "yes"; then
-    if compile_prog "" "-luring" "libaio io_uring" ; then
+    if compile_prog "" "-laio -luring" "libaio io_uring" ; then
       libaio=yes
-      LIBS="-luring $LIBS"
+      LIBS="-laio -luring $LIBS"
     else
       feature_not_found "libaio io_uring" ""
     fi
```

3、./configure --build-static --cc=/home/akon/src/akon/BiscuitOS/output/linux-5.2-aarch/aarch64-linux-gnu/aarch64-linux-gnu/bin/aarch64-linux-gnu-gcc --enable-libaio-uring

3、make

> 测试

./fio -bs=4k -ioengine=io_uring -iodepth=1 -direct=1 -rw=randrw -time_based  -runtime=600  -refill_buffers -norandommap -randrepeat=0 -group_reporting -name=fio-randrw-lat --size=1G -filename=./fio_test_file --output=fio_io_uring_result

```
ioengine：io引擎选择io_uring

direct：绕过文件系page cache

rw=randrw：测试随机读写

-time-base：默认不指定在指定大小文件测试完毕之后，将结runtime时间，加上会一定执行完对应时间

-runtime：指定运行时长

-norandommap ：不指定的话，fio会随机读写完一个文件的所有block，选择下一个block会查询当前已经读写的block，加上的话有些block可能会多次被选中

-randrepeat： 对于随机IO负载，配置生成器的种子，使得路径是可以预估的，使得每次重复执行生成的序列是一样的 

-group-reporting： 如果‘numjobs’设置的话，我们感兴趣的可能是打印group的统计值，而不是一个单独的job。这在‘numjobs’的值很大时，一般是设置为true的，可以减少输出的信息量。如果‘group_reporting’设置的话，fio将会显示最终的per-groupreport而不是每一个job都会显示。 

-name：指定线程名字

iodepth：应用同一时刻发送多少io给内核
```

./fio -bs=4k -ioengine=io_uring -iodepth=1 -direct=1 -rw=randrw -time_based  -runtime=600  -refill_buffers -norandommap -randrepeat=0 -group_reporting -name=fio-randrw-lat --size=1G -filename=./fio_test_file --output=fio_io_uring_result  -io_submit_mode=offload -hipri -sqthread_poll -sqthread_poll-cpu

```
 -io_submit_mode=offload：开启submission offload 模式
 -hipri：在块设备层支持io polled模式
 -fixedbufs：一开始就映射所有io，能有效的降低io延时
 -registerfiles：
 -sqthread_poll：
 -sqthread_poll-cpu：
```



- 3、测试结果分析

https://blog.csdn.net/feilianbb/article/details/50497845