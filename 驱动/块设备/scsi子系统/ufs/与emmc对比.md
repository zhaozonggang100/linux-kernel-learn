
引用：
```
https://blog.csdn.net/shenjin_s/article/details/79761425?utm_medium=distribute.pc_relevant_download.none-task-blog-blogcommendfrombaidu-3.nonecase&depth_1-utm_source=distribute.pc_relevant_download.none-task-blog-blogcommendfrombaidu-3.nonecas

https://blog.csdn.net/LUOHUATINGYUSHENG/article/details/94624348?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param
```


### 1、速度

![](.\picture\ufs与emmc速度对比.png)

![](.\picture\ufs-emmc-speed.jpg)

![](.\picture\ufs-emmc-speed1.jpg)

### 2、电源

- UFS

需要三路电源信号： VCC，VCCQ和VCCQ2 

- EMMC

需要两路电源信号：VCC，VCCQ

### 3、传输

- UFS

两组差分数据线+一条时钟线

- EMMC

8条数据总线复用并行

### 4、优点

```
1）抗EMI和串扰
如果从差分导体外部引入EMI或串扰，则将其同等地添加到反相和非反相信号。
因此，接收器电路极大地降低了干扰或串扰的幅度。
2）降低EMI和串扰
差分对中的两个信号产生（理想情况下）幅度相等但极性相反的电磁场。
这适用于UFS卡，并确保两个导体的发射在很大程度上相互抵消。
3）更好的信噪比
UFS卡中的差分信号可以使用较低的电压，并且由于提高了抗噪声性能，仍然保持足够的信噪比（SNR），而SD卡中的单端信号需要稳定的高电压以确保足够的SNR。
4）接收器电路的复杂性降低
将差分信号集成到UFS卡中，确定逻辑状态更简单，
就像比较反相和非反相信号的电压一样。
然而，在SD卡的单端系统中，接收器电路更复杂，应考虑参考电压的值以及变化和容差。
5）专业物理/链接和命令层的优点
```

