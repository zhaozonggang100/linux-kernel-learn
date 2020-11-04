https://source.android.google.cn/devices/architecture/kernel/bpf

http://www.caveman.work/2019/01/29/eBPF-on-Android/

### 1、bcc支持

https://www.sohu.com/a/409335977_467784

https://blog.csdn.net/buhui912/article/details/106118099

https://blog.csdn.net/buhui912/article/details/106118056

### 2、adeb

https://www.androidos.net.cn/android/10.0.0_r6/raw/external/adeb/androdeb

https://source.android.google.cn/devices/tech/debug/native_stack_dump?hl=zh-cn

> 1、简介

用来在Android中跑debian环境，来支持bcc开发环境

> 2、运行条件

- 1、目标机器

1、ARM64

2、Android N or later

3、支持adb root

4、data分区有2GB以上剩余空间

- 2、主机

1、ubuntu或debian的比较新的版本

2、拥有4GB的free 内存

3、debootstrap and qemu-debootstrap packages

```
sudo apt-get install qemu-user-static debootstrap
```

> 3、快速开始

1、进入安卓源码external目录，然后进入adeb目录

2、设置当前用户环境变量

`export ADEB_REPO_URL="github.com/joelagnel/adeb/"`

3、安装adeb到自己的主机

普通安装（只安装base包）：`adeb prepare`

全量安装：

```
adeb prepare --full  // 这样可能会因为内核版本和机器上的不一致而警告，可以忽略
adeb prepare --full --kernelsrc /path/to/kernel-source/   // 指定自己的kernel路径，这样adeb中的kernel版本和机器本地的一致，开机不会报警告

要使机器本地的内核和adeb中的内核版本一致，需要对自己本地的内核做如下配置：
You need kernel 4.9 or newer. Anything less needs backports. Your kernel needs
to be built with the following config options at the minimum:
​```
CONFIG_KPROBES=y
CONFIG_KPROBE_EVENT=y
CONFIG_BPF_SYSCALL=y
​```
Optionally,
​```
CONFIG_UPROBES=y
CONFIG_UPROBE_EVENTS=y
​```
Additionally, for the criticalsection BCC tracer to work, you need:
​```
CONFIG_DEBUG_PREEMPT=y
CONFIG_PREEMPTIRQ_EVENTS=y
​```
```

4、进入adeb  shell环境，登录到开发板的debian上

如果要安装bcc，需要先执行如下命令：`adeb prepare --build --bcc`

```
如果该过程报如下错：
fatal: unable to access 'https://github.com/libbpf/libbpf.git/': Could not resolve host: github.com
fatal: clone of 'https://github.com/libbpf/libbpf.git' into submodule path '/bcc-master/src/cc/libbpf' failed
Failed to clone 'src/cc/libbpf' a second time, aborting

原因：
github该仓库访问总是出错，需要手动clone

解决办法：

1、修改buildstrap
 diff --git a/buildstrap b/buildstrap
index b33b71e..4bc4cf3 100755
--- a/buildstrap
+++ b/buildstrap
@@ -15,7 +15,8 @@ SKIP_DEVICE=$8                # Skip any device-specific stages
 VARIANT="--variant=minbase"
 
 time qemu-debootstrap --arch $ARCH --include=$PACKAGES $VARIANT \
-       $DISTRO $OUT_TMP http://deb.debian.org/debian/
+       $DISTRO $OUT_TMP https://mirror.tuna.tsinghua.edu.cn/debian/
+#$DISTRO $OUT_TMP http://deb.debian.org/debian/
 
 # Some reason debootstrap leaves these mounted
 umount $OUT_TMP/proc/sys/fs/binfmt_misc || true
@@ -61,7 +62,8 @@ echo "nameserver 4.2.2.2" > $OUT_TMP/etc/resolv.conf
 
 # Clone BCC if needed
 if [ $INSTALL_BCC -eq 1 ]; then
-       git clone https://github.com/iovisor/bcc.git $TDIR/debian/bcc-master
+       #git clone https://github.com/iovisor/bcc.git $TDIR/debian/bcc-master
+       cp -rf $spath/bcc/bcc-master $TDIR/debian/;
        cp $spath/bcc/build-bcc.sh $TDIR/debian/bcc-master/;
 fi

2、重新执行
adeb prepare --build --bcc
```



需要先执行下面命令：`./androdeb prepare`

单个设备连接主机：`adeb shell`

多个设备连接主机：`adeb -s xxx shell`

预编译好的bcc工具：`  /usr/share/bcc/tools `

5、退出adbeb shell

`CTRL+D`



---

<font color="red">上面五个步骤及完成了adeb环境的基本搭建</font>

---



6、用完想移除adeb

`adeb  remove`

7、更新设备已经存在一个adeb

`adeb git-pull`

8、为了方便部署，可以将配置好的adeb的rootfs环境打包，部署到其他机器

```
1、打包
	adeb prepare --buildtar xxx/
	上面命令会将已经部署并安装了必要升级包的adeb环境打包到xxx路径下的adeb-fs.tgz文件
2、快速部署到其他机器
	adeb prepare --archive xxx/adeb-fs.tgz
```

9、QEMU模拟adeb环境，不需要真实的安卓设备

```
1、准备一个ext4的磁盘镜像
2、adeb prepare --build-image /path/to/ext4.img
```

10、apt安装包慢，更新apt源

```
1、adeb shell
2、替换/etc/apt/sources.list中的apt源地址为https://mirror.tuna.tsinghua.edu.cn/debian
3、apt-get update
```

11、执行adeb命令编译安装bcc慢

```
在安装源码中替换adeb命令编译安装的源（external/adeb/buildstrap）

 17 time qemu-debootstrap --arch $ARCH --include=$PACKAGES $VARIANT \
 18     $DISTRO $OUT_TMP https://mirror.tuna.tsinghua.edu.cn/debian/
 19 #$DISTRO $OUT_TMP http://deb.debian.org/debian/
```



