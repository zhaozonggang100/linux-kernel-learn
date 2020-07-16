1、映射物理页到连续虚拟地址

```c
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
EXPORT_SYMBOL(vmap);
```

2、释放连续虚拟地址对应的物理页，不能用于中断上下文

```c
/**
 *  vunmap  -  release virtual mapping obtained by vmap()
 *  @addr:      memory base address
 *
 *  Free the virtually contiguous memory area starting at @addr,
 *  which was created from the page array passed to vmap().
 *
 *  Must not be called in interrupt context.
 */
void vunmap(const void *addr)
{
    BUG_ON(in_interrupt());
    might_sleep();
    if (addr)
        __vunmap(addr, 0);
}
EXPORT_SYMBOL(vunmap);
```