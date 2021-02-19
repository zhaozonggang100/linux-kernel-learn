### 1、info

- ctxt：所有cpu上发生的上下文切换次数，与vmstat显示的`cs`对应

```c++
// https://blog.csdn.net/weixin_30235225/article/details/98615471
unsigned long long nr_context_switches(void)
{
        int i;
        unsigned long long sum = 0;

        for_each_possible_cpu(i)
                sum += cpu_rq(i)->nr_switches;

        // sum就是从开机到现在所有cpu上下文切换的次数
        return sum;
}
```

### 2、kernel create

**file**：`fs/proc/stat.c`

```c++
static int __init proc_stat_init(void)
{
        proc_create("stat", 0, NULL, &proc_stat_operations);
        return 0;
}
fs_initcall(proc_stat_init);
```