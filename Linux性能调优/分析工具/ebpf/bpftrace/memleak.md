### 1、概要

使用bpftrace编写一个light level的memleak tools

运行bt脚本之前需要打开如下选项：

`echo 1 > /proc/sys/net/core/bpf_jit_enable`

### 2、code

```c
#!/usr/bin/env bpftrace

BEGIN
{
        printf("Bpftrce detect kernel memleak ... Hit ctrl-c to end.\n");
}

tracepoint:kmem:kmalloc
{
        printf("%s\n", kstack);

        //if (!@kmem_addr) {
        //      @alloc_stack = kstack;
        //      @kmem_addr = args->ptr;
        //}
}

//tracepoint:kmem:kfree
///@kmem_addr/
//{
        //delete(@kmem_addr);
        //delete(@alloc_stack);
//}


//interval:s:30
//{
        //if (@kmem_addr) {
        //      printf("kmem_addr: 0x%lx\n", @kmem_addr);
        //      printf("%s\n", @alloc_stack);
        //}
//
```

### 3、issue

5.4之前的内存tracepoint获取不到全部的stack

https://github.com/iovisor/bcc/issues/2332

https://github.com/torvalds/linux/commit/fe8d9571dc50232b569242fac7ea6332a654f186

https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=bd98c81346468fc2f86aeeb44d4d0d6f763a62b7

![x86](D:\akon\note\akon\linux-kernel-learn\Linux性能调优\分析工具\ebpf\bpftrace\tracepoint_stack_issue.png)

