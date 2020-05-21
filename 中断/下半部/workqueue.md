参考：

<https://www.cnblogs.com/arnoldlu/p/8659988.html>



### 1、节能wq

配置了CONFIG_WQ_POWER_EFFICIENT_DEFAULT表示开启wq节能



### 2、wq特性

1、cpumask  

/sys/devices/virtual/workqueue/cpumask代表执行queue_work时如果不使能节能wq，默认该work会被wq调度被中断的cpu的handler函数中执行queue_work的cpu上，及时该cpu处于idle状态，但也不一定，需要判断cpumask是否将local cpu置0，即使置0也可能被调度到别的cpu上，需要看调度器的行为

cpumask只有在执行queue_work时才生效，如果执行的是queue_work_on或者queue_delay_work_on时失效