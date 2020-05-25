### 1、oops调试  
参考：  
https://blog.csdn.net/kris_fei/article/details/77936620
现场：  

```
04-21 16:43:06.015     0     0 W k       : Call trace:
04-21 16:43:06.015     0     0 W k       : [<ffffff80080882c4>] dump_backtrace+0x0/0x1c0
04-21 16:43:06.015     0     0 W k       : [<ffffff8008088498>] show_stack+0x14/0x1c
04-21 16:43:06.015     0     0 W k       : [<ffffff80083234f4>] dump_stack+0x94/0xb4
04-21 16:43:06.015     0     0 W k       : [<ffffff80081acf94>] dump_header.isra.12+0x3c/0x6c
04-21 16:43:06.015     0     0 W k       : [<ffffff80081654c0>] oom_kill_process+0x8c/0x3dc
04-21 16:43:06.015     0     0 W k       : [<ffffff8008165ac0>] out_of_memory+0x234/0x2a8
04-21 16:43:06.015     0     0 W k       : [<ffffff800816a410>] __alloc_pages_nodemask+0x6e4/0x820
04-21 16:43:06.015     0     0 W k       : [<ffffff80081a3000>] allocate_slab+0xb4/0x244
04-21 16:43:06.015     0     0 W k       : [<ffffff80081a439c>] ___slab_alloc.isra.68.constprop.75+0x58c/0x600
04-21 16:43:06.015     0     0 W k       : [<ffffff80081a4434>] __slab_alloc.isra.69.constprop.74+0x24/0x34
04-21 16:43:06.015     0     0 W k       : [<ffffff80081a4720>] kmem_cache_alloc+0x94/0x210
04-21 16:43:06.015     0     0 W k       : [<ffffff8008327be8>] __radix_tree_preload+0x70/0xec
04-21 16:43:06.015     0     0 W k       : [<ffffff8008327ca4>] radix_tree_maybe_preload+0x10/0x30
04-21 16:43:06.015     0     0 W k       : [<ffffff800819d164>] __read_swap_cache_async+0xa4/0x1f0
04-21 16:43:06.015     0     0 W k       : [<ffffff800819d2c4>] read_swap_cache_async+0x14/0x34
04-21 16:43:06.015     0     0 W k       : [<ffffff800819d464>] swapin_readahead+0x180/0x19c
04-21 16:43:06.015     0     0 W k       : [<ffffff800818e0ec>] handle_mm_fault+0xa20/0xe4c
04-21 16:43:06.015     0     0 W k       : [<ffffff8008095098>] do_page_fault+0x17c/0x320
04-21 16:43:06.015     0     0 W k       : [<ffffff8008095280>] do_translation_fault+0x44/0xc0
04-21 16:43:06.015     0     0 W k       : [<ffffff8008080b68>] do_mem_abort+0x40/0x9c
04-21 16:43:06.015     0     0 W k       : Exception stack(0xffffffc177887af0 to 0xffffffc177887c20)
04-21 16:43:06.015     0     0 W k       : 7ae0:                                   0000000000000018 0000008000000000
04-21 16:43:06.015     0     0 W k       : 7b00: ffffffc177887cc0 ffffff8008321d90 ffffffc177887b70 ffffff80085db4f4
04-21 16:43:06.015     0     0 W k       : 7b20: ffffffc0732aaf00 000000000003eae0 ffffffc05ebf2400 000000000003eae0
04-21 16:43:06.015     0     0 W k       : 7b40: 000000000003eae0 0000000000003eae ffffff8009a3a838 00000004426d2920
04-21 16:43:06.015     0     0 W k       : 7b60: 000000020000006f ffffffbdc3204780 ffffffc177887ba0 ffffff80085db5fc
04-21 16:43:06.015     0     0 W k       : 7b80: ffffffc1788dee00 ffffffc0732aaf00 00000075aa500420 ffffffc177887d90
04-21 16:43:06.016     0     0 W k       : 7ba0: 0000000000000010 0000000000000000 0000000000000008 00000075aa500438
04-21 16:43:06.016     0     0 W k       : 7bc0: 00000075aa500420 ffffffc066232c01 ffffff8008c31258 ffffffc1778877e0
04-21 16:43:06.016     0     0 W k       : 7be0: 0000000000000a00 0000069b67450a3e 0000000001312d00 0000000000000e64
04-21 16:43:06.016     0     0 W k       : 7c00: ffffffffffffffff 0000c3746ae96a6f 0000007e9deebcd8 0000007e9de89f24
04-21 16:43:06.016     0     0 W k       : [<ffffff80080825cc>] el1_da+0x24/0x84
04-21 16:43:06.016     0     0 W k       : [<ffffff80081c0b94>] SyS_pselect6+0x154/0x1d0
04-21 16:43:06.016     0     0 W k       : [<ffffff8008082fb0>] el0_svc_naked+0x24/0x28
```
1、方法1：gdb调试vmlinux  
```
	xxx-gdb  vmlinux
		l *do_mem_abort+0x40
	显示arch/arm64/mm/fault.c:599，为缺页异常引起的
```

2、方法2：反汇编（disassemble）  
3、方法3  



### 2、oops系统配置

/proc/sys/kernel/panic_on_oops：当发生oops系统立刻重启