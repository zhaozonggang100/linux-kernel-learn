### 1、普通Linux平台才看cpu信息

cat /proc/cpuinfo

### 2、android平台查看cpu信息

/sys/devices/system/cpu/

### 3、查看进程运行在哪个核上

ps -o -psr -p pid

