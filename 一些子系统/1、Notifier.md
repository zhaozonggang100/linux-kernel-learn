###　１、cpu notifier子系统  

```
在rcu子系统中会向cpu notifier子系统注册CPU_UP_PREPARE事件，然后在rcu_cpu_notifier中处理cpu事件
```

