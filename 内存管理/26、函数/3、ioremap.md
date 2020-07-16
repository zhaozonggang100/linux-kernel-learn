### 1、概述

### 2、使用

1、分配region标记对应的物理内存已经busy

```c
#define request_mem_region(start,n,name) __request_region(&iomem_resource, (start), (n), (name), 0)

/**
 * __request_region - create a new busy resource region
 * @parent: parent resource descriptor
 * @start: resource start address
 * @n: resource region size
 * @name: reserving caller's ID string
 * @flags: IO resource flags
 */
struct resource * __request_region(struct resource *parent,
                   resource_size_t start, resource_size_t n,
                   const char *name, int flags)
{
    DECLARE_WAITQUEUE(wait, current);
    struct resource *res = alloc_resource(GFP_KERNEL);

    if (!res) {
        return NULL;
    }    

    res->name = name;
    res->start = start;
    res->end = start + n - 1; 

    write_lock(&resource_lock);

    for (;;) {
        struct resource *conflict;

        res->flags = resource_type(parent) | resource_ext_type(parent);
        res->flags |= IORESOURCE_BUSY | flags;
        res->desc = parent->desc;

        conflict = __request_resource(parent, res);
        if (!conflict) {
            break;
        }
        if (conflict != parent) {
            if (!(conflict->flags & IORESOURCE_BUSY)) {
                parent = conflict;
                continue;
            }
        }
        if (conflict->flags & flags & IORESOURCE_MUXED) {
            add_wait_queue(&muxed_resource_wait, &wait);
            write_unlock(&resource_lock);
            set_current_state(TASK_UNINTERRUPTIBLE);
            schedule();
            remove_wait_queue(&muxed_resource_wait, &wait);
            write_lock(&resource_lock);
            continue;
        }
        /* Uhhuh, that didn't work out.. */
        free_resource(res);
        res = NULL;
        break;
    }
    write_unlock(&resource_lock);
    return res;
}
EXPORT_SYMBOL(__request_region);
```

2、使用ioremap进行地址映射

```c
// file:arch/arm64/include/asm/io.h
#define ioremap(addr, size)     __ioremap((addr), (size), __pgprot(PROT_DEVICE_nGnRE))

// file：arch/arm64/mm/ioremap.c
void __iomem *__ioremap(phys_addr_t phys_addr, size_t size, pgprot_t prot)
{
    return __ioremap_caller(phys_addr, size, prot,
                __builtin_return_address(0));
}
EXPORT_SYMBOL(__ioremap);

static void __iomem *__ioremap_caller(phys_addr_t phys_addr, size_t size,
                      pgprot_t prot, void *caller)
{
    unsigned long last_addr;
    unsigned long offset = phys_addr & ~PAGE_MASK;
    int err;
    unsigned long addr;
    struct vm_struct *area;

    /*  
     * Page align the mapping address and size, taking account of any
     * offset.
     */
    phys_addr &= PAGE_MASK;
    size = PAGE_ALIGN(size + offset);

    /*  
     * Don't allow wraparound, zero size or outside PHYS_MASK.
     */
    last_addr = phys_addr + size - 1;
    if (!size || last_addr < phys_addr || (last_addr & ~PHYS_MASK))
        return NULL;

    /*  
     * Don't allow RAM to be mapped.
     */
    if (WARN_ON(pfn_valid(__phys_to_pfn(phys_addr))))
        return NULL;

    area = get_vm_area_caller(size, VM_IOREMAP, caller);
    if (!area)
        return NULL;
    addr = (unsigned long)area->addr;
    area->phys_addr = phys_addr;

    err = ioremap_page_range(addr, addr + size, phys_addr, prot);
    if (err) {
        vunmap((void *)addr);
        return NULL;
    }   

    return (void __iomem *)(offset + addr);
}
```

3、使用完释放region

```c
// file:arch/arm64/include/asm/io.h
#define release_mem_region(start,n) __release_region(&iomem_resource, (start), (n))

/**
 * file：kernel/resource.c
 * __release_region - release a previously reserved resource region
 * @parent: parent resource descriptor
 * @start: resource start address
 * @n: resource region size
 *
 * The described resource region must match a currently busy region.
 */
void __release_region(struct resource *parent, resource_size_t start,
            resource_size_t n)
{
    struct resource **p;
    resource_size_t end;

    p = &parent->child;
    end = start + n - 1;
                    
    write_lock(&resource_lock);

    for (;;) {
        struct resource *res = *p;
        
        if (!res)
            break;
        if (res->start <= start && res->end >= end) {
            if (!(res->flags & IORESOURCE_BUSY)) {
                p = &res->child;
                continue;
            }
            if (res->start != start || res->end != end)
                break;
            *p = res->sibling;
            write_unlock(&resource_lock);
            if (res->flags & IORESOURCE_MUXED)
                wake_up(&muxed_resource_wait);
            free_resource(res);
            return;
        }
        p = &res->sibling;
    }

    write_unlock(&resource_lock);

    printk(KERN_WARNING "Trying to free nonexistent resource "
        "<%016llx-%016llx>\n", (unsigned long long)start,
        (unsigned long long)end);
}
EXPORT_SYMBOL(__release_region);
```



4、释放页映射

```c
// file:arch/arm64/include/asm/io.h
#define iounmap             __iounmap

void __iounmap(volatile void __iomem *io_addr)
{
    unsigned long addr = (unsigned long)io_addr & PAGE_MASK;

    /*
     * We could get an address outside vmalloc range in case
     * of ioremap_cache() reusing a RAM mapping.
     */
    if (VMALLOC_START <= addr && addr < VMALLOC_END)
        vunmap((void *)addr);
}
EXPORT_SYMBOL(__iounmap);
```

