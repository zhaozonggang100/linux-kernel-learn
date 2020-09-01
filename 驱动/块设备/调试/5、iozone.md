### 1、概述

- 1、官网

   http://www.iozone.org/ 
   
- 2、参考

```
https://www.jianshu.com/p/faf82e400aa6
```



### 2、ubuntu安装

- 1、sudo apt-get insyall -y
- 2、sudo apt-get install iozone3



### 3、Android静态交叉编译

- 1、download source 在`http://www.iozone.org/ `

- 2、将源码解压到android/external中
- 3、进入源码src目录，执行mm（前提是已经source和lunch过了）
- 4、输出文件在out/xxx/xxx/system/bin中



### 4、测试实例

- 1、测试1

```
命令：iozone -a -n 512m -g 4g -i 0 -i 1 -i 1 -f /tmp/iozone -Rb ./iozone.xls
	-a：全面测试，每次读写的块大小自动增加
	-n：测试文件最小512m
	-g：测试文件最大4g
	-i：指定测试项，0--写，1--读
	-f：测试文件，测试完会删除
	-R：产生execl到标准输出
	-b：指定输出的execl文件
	-I：对测试文件跨越缓存测试
	-s：指定测试文件的大小，最好大于内存大小
```



