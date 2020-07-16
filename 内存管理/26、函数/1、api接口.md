1、验证物理页经过PAGE_SHIFT偏移之后对应的页帧是否有效的
```c
#ifdef CONFIG_HAVE_ARCH_PFN_VALID		// 需要使能验证页帧有效的config
int pfn_valid(unsigned long pfn)
{
    phys_addr_t addr = __pfn_to_phys(pfn);	// 页帧转物理地址

    if (__phys_to_pfn(addr) != pfn)
        return 0;
    
    return memblock_is_map_memory(__pfn_to_phys(pfn));	// memblock是内核初始化时候的分配器

}
EXPORT_SYMBOL(pfn_valid);
#endif
```



