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

2、映射连续的虚拟地址到物理页

```
/**
 *  vmap  -  map an array of pages into virtually contiguous space
 *  @pages:     array of page pointers
 *  @count:     number of pages to map
 *  @flags:     vm_area->flags
 *  @prot:      page protection for the mapping
 *
 *  Maps @count pages from @pages into contiguous kernel virtual
 *  space.
 */
void *vmap(struct page **pages, unsigned int count,
        unsigned long flags, pgprot_t prot)
{
    struct vm_struct *area;
    unsigned long size;     /* In bytes */

    might_sleep();

    if (count > totalram_pages)
        return NULL;

    size = (unsigned long)count << PAGE_SHIFT;
    area = get_vm_area_caller(size, flags, __builtin_return_address(0));
    if (!area)
        return NULL;

    if (map_vm_area(area, prot, pages)) {
        vunmap(area->addr);
        return NULL;
    }

    return area->addr;
}
```

