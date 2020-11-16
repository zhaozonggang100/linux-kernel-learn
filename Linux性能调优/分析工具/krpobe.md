# 1、概述

随着Linux进入4.x阶段，ebpf的出现，高效率、高安全的全套Linux性能监测工具集提供给大家使用，不过经过本人亲测，需要5.4以后的内核算比较稳定的支持ebpf，如此高效的ebpf也利用了kprobe实现他的动态插桩，具体kprobe是什么，大家自己百度吧。

# 2、内核源码

`samples/kprobes`：kprobe的例子

` Documentation/kprobes.txt `：kprobe的介绍

# 3、kprobe注册方法

### 1、通过debugfs注册

- 1、 cd /sys/kernel/debug/tracing 		// 这是ftrace的天下，kprobe只是ftrace的一个接口，还有tracepoint、 function trace 等，ftrace的输出由ftrace子系统限定了，对于具体的个人需求需要自己定制
- 2、 echo "p:sys_write_event sys_write" > kprobe_events   //  当前目录下的events下，新增一个kprobes目录 

```
我们注册的kprobe时间生效了，那么“p:sys_write_event sys_write”是什么意思呢？首先p表示我们要注册一个kprobe，如果要注册retprobe，此处应为r；sys_write_event表示这个kprobe叫什么名字；sys_write表示我们的插入点在哪里。那么，“p:sys_write_event sys_write”的语义就很明显了：在函数sys_write处插入了一个kprobe点，这个点的名字叫做sys_write_event。
```

- 3、使能kprobe

```
cd  /sys/kernel/debug/tracing/events/kprobes/events/sys_write_event
echo 1 > enable
cd /sys/kernel/debug/tracing //查看trace文件的输出
cat trace
```

- 4、撤销kprobe

```
cd /sys/krenel/debug/tracing/events/sys_write_event
echo 0 > enable[首先先关闭kprobe]
cd ../../..
echo "-:kprobes/sys_write_event" >> kprobe_events
```

### 2、通过写kprobe模块

# 4、数据结构

```c
struct kprobe {
    //哈希链表, 被静态全局变量kprobe_table管理, 每个被监测地址作为索引
    //如果一个地址存在多个kprobe则该哈希节点会用aggregate节点替代 
    struct hlist_node hlist;
    /* list of kprobes for multi-handler support */
    struct list_head list;
    /*count the number of times this probe was temporarily disarmed */
    //因断点指令不能重入处理, 当多个kprobe一起触发时会放弃执行后面的probe, 同时该计数增加 
    unsigned long nmissed;
    /* location of the probe point */
    //观察点对应的地址, 用户在调用注册接口时可以指定地址, 也可以传入函数名让内核自己查找 
    kprobe_opcode_t *addr;
    /* Allow user to indicate symbol name of the probe point */
    //观察点对应的函数名, 在注册kprobe时会将其翻译为十六进制地址并修改addr 
    const char *symbol_name;
    /* Offset into the symbol */
    //相对于入口点地址的偏移, 会在计算addr以后再加上offset得到最终的addr
    unsigned int offset;
    /* Called before addr is executed. */
    kprobe_pre_handler_t pre_handler;
    /* Called after addr is executed, unless... */
    kprobe_post_handler_t post_handler;
    /*  
     * ... called if executing addr causes a fault (eg. page fault).
     * Return 1 if it handled fault, otherwise kernel will see it.
     */
    kprobe_fault_handler_t fault_handler;
    /*  
     * ... called if breakpoint trap occurs in probe handler.
     * Return 1 if it handled break, otherwise kernel will see it.
     */
    kprobe_break_handler_t break_handler;
    /* Saved opcode (which has been replaced with breakpoint) */
    kprobe_opcode_t opcode;
    /* copy of the original instruction */
    struct arch_specific_insn ainsn;
    /*  
     * Indicates various status flags.
     * Protected by kprobe_mutex after this kprobe is registered.
     */
    u32 flags;
};

/*
 * This struct defines the way the registers are stored on the stack during an
 * exception. Note that sizeof(struct pt_regs) has to be a multiple of 16 (for
 * stack alignment). struct user_pt_regs must form a prefix of struct pt_regs.
 */
// 体系架构相关，这里是分析aarch64
struct pt_regs {
    union {
        struct user_pt_regs user_regs;
        struct {
            u64 regs[31];
            u64 sp;
            u64 pc;
            u64 pstate;
        };
    };
    u64 orig_x0;
#ifdef __AARCH64EB__
    u32 unused2;
    s32 syscallno;
#else
    s32 syscallno;
    u32 unused2;
#endif

    u64 orig_addr_limit;
    u64 unused; // maintain 16 byte alignment
    u64 stackframe[2];
};

/*
 * Function-return probe -
 * Note:
 * User needs to provide a handler function, and initialize maxactive.
 * maxactive - The maximum number of instances of the probed function that
 * can be active concurrently.
 * nmissed - tracks the number of times the probed function's return was
 * ignored, due to maxactive being too low.
 *
 */
struct kretprobe {
    struct kprobe kp; 
    kretprobe_handler_t handler;
    kretprobe_handler_t entry_handler;
    int maxactive;
    int nmissed;
    size_t data_size;
    struct hlist_head free_instances;
    raw_spinlock_t lock;
};
```

