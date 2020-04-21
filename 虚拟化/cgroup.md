### 1、目的
1、实现Linux进程分组化管理  
2、限制进程可以使用的资源数量，限制进程最大使用内存  
3、进程组的优先级控制，比如为某个进程组分配特定的cpu share  
4、记录进程组使用的资源数量，比如记录某个进程cpu的使用时间  
5、进程组隔离，通过namespace达到隔离  
6、进程组控制，将进程组挂起或者恢复  
	
### 2、对应的proc系统文件  
1、/proc/cgroups：运行中的内核可用的cgroup  
2、/proc/mounts：查看已经挂载的cgroup子系统  
			
### 3、cgroup文件系统  
cgroup提供了一个cgroup虚拟文件系统，要使用cgroup，必须挂载cgroup文件系统  
	
### 4、cgroup模型  
继承：子进程会继承父进程所属的cgroup分组  

### 5、数据结构
task_struct-->css_set-->cgroup_subsys_state-->cgroup  		
task_struct-->cg_list  
		
cgroup_subsys_state：一个cgroups子系统用一个cgroup_subsys_state结构表示，这是一个子系统状态的集合  
css_set：一个css_set对应多个子系统  
cgroup：进程属于哪个组，一个进程可以属于多个组  
cg_list：将所有具有相同css_set的进程连接起来  

###	6、涉及到的概念  
subsystems：一个子系统就是一个资源控制器，子系统只有被挂载才能控制进程使用资源  
> 1、blkio  
				
块设备输入、输出限制  
控制进程io的优先级  
作为CFQ IO调度器的一部分  
查看当前系统是否是CFQ IO调度器：/sys/class/block/sda/queue/scheduler  
			
> 2、cpu控制器  

使用调度程序提供对cpu的cgroup任务访问  

> 3、cpuacct  
				
自动生生cgroup中任务使用的cpu资源报告  
			
> 4、cpuset  

为cgroup中的任务分配独立CPU（多核系统）和内存节点  
			
> 5、devices  

可允许或者拒绝任务对设备的访问  

> 6、freezer  

挂起或者恢复cgroup中的任务  
			
> 7、memory  

设定cgroup中任务使用的内存限制，并自动生成任务使用的内存资源报告  

> 8、net_cls  

使用等级识别符(classid)标记网络数据包，可允许 Linux 流量控制程序（tc）识别从具体 cgroup 中生成的数据包。  
			
> 9、net_prio  

允许管理员动态的通过各种应用程序设置网络传输的优先级，类似于socket 选项的SO_PRIORITY，但它有它自身的优势。  

> 10、HugeTLB  

HugeTLB页的资源控制功能  
			
hierarchies：继承体系（或层次体系），cgroups是按照层次体系的关系进行组织的control groups：一组按照某种标准划分的进程，进程可以从一个control groups中迁移到另一个control groups  
		
tasks：cgroups中的一个进程  
		
限制：
```
1、一个子系统不能被附加到多个继承体系中
2、每当在系统中创建一个继承体系的时候，会默认创建一个control groups，并且称control groups为root cgroup，此时整个系统中的tasks都属于这个root cgroup
3、任何进程fork子进程的时候，子进程会自动继承父进程的control groups，成为其中的一员，但是子进程可以迁移到别的控制组
```