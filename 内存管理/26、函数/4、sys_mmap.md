### 1、概述

创建文件映射、匿名页映射、巨页映射

### 2、函数详解

```c
 // arch/arm64/kernel/sys.c，体系相关
 29 asmlinkage long sys_mmap(unsigned long addr, unsigned long len,          
 30              unsigned long prot, unsigned long flags,
 31              unsigned long fd, off_t off)
 32 {
     	// 判断偏移是否页对齐，匿名映射是0，也算对齐
 33     if (offset_in_page(off) != 0)
 34         return -EINVAL;
 35 
 36     return sys_mmap_pgoff(addr, len, prot, flags, fd, off >> PAGE_SHIFT);
 37 }

 // 体系无关，mm/mmap.c
1497 SYSCALL_DEFINE6(mmap_pgoff, unsigned long, addr, unsigned long, len,
1498         unsigned long, prot, unsigned long, flags,
1499         unsigned long, fd, unsigned long, pgoff)
1500 {
1501     struct file *file = NULL;
1502     unsigned long retval;
1503 
1504     if (!(flags & MAP_ANONYMOUS)) {
    		 // 非匿名映射，也就是文件映射
1505         audit_mmap_fd(fd, flags);
    		 // 从文件fd获取file结构
1506         file = fget(fd);
1507         if (!file)
1508             return -EBADF;
1509         if (is_file_hugepages(file))
1510             len = ALIGN(len, huge_page_size(hstate_file(file)));
1511         retval = -EINVAL;
1512         if (unlikely(flags & MAP_HUGETLB && !is_file_hugepages(file)))
1513             goto out_fput;
1514     } else if (flags & MAP_HUGETLB) {
    		 // 标准巨页映射
1515         struct user_struct *user = NULL;
1516         struct hstate *hs;
1517 
1518         hs = hstate_sizelog((flags >> MAP_HUGE_SHIFT) & SHM_HUGE_MASK);
1519         if (!hs)
1520             return -EINVAL;
1521 
1522         len = ALIGN(len, huge_page_size(hs));
1523         /*
1524          * VM_NORESERVE is used because the reservations will be
1525          * taken when vm_ops->mmap() is called
1526          * A dummy user value is used because we are not locking
1527          * memory so no accounting is necessary
1528          */
1529         file = hugetlb_file_setup(HUGETLB_ANON_FILE, len,
1530                 VM_NORESERVE,
1531                 &user, HUGETLB_ANONHUGE_INODE,
1532                 (flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
1533         if (IS_ERR(file))
1534             return PTR_ERR(file);
1535     }
1536 
1537     flags &= ~(MAP_EXECUTABLE | MAP_DENYWRITE);
1538 
1539     retval = vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);
1540 out_fput:
1541     if (file)
1542         fput(file);
1543     return retval;
1544 }

285 unsigned long vm_mmap_pgoff(struct file *file, unsigned long addr,
286     unsigned long len, unsigned long prot,
287     unsigned long flag, unsigned long pgoff)
288 {
289     unsigned long ret;
290     struct mm_struct *mm = current->mm;
291     unsigned long populate;
292 
293     ret = security_mmap_file(file, prot, flag);
294     if (!ret) {
295         down_write(&mm->mmap_sem);
296         ret = do_mmap_pgoff(file, addr, len, prot, flag, pgoff,
297                     &populate);
298         up_write(&mm->mmap_sem);
    		// 调用者要求将页锁定在内存中，或者要求填充页表并进行阻塞
299         if (populate)
300             mm_populate(ret, populate);
301     }
302     return ret;
303 }

// include/linux/mm.h
1915 static inline unsigned long
1916 do_mmap_pgoff(struct file *file, unsigned long addr,
1917     unsigned long len, unsigned long prot, unsigned long flags,
1918     unsigned long pgoff, unsigned long *populate)
1919 {
1920     return do_mmap(file, addr, len, prot, flags, 0, pgoff, populate);
1921 }

1339 /*
1340  * The caller must hold down_write(&current->mm->mmap_sem).
1341  */
// mm/mmap.c，创建内存区域映射的核心函数
1342 unsigned long do_mmap(struct file *file, unsigned long addr,
1343             unsigned long len, unsigned long prot,
1344             unsigned long flags, vm_flags_t vm_flags,
1345             unsigned long pgoff, unsigned long *populate)
1346 {   
1347     struct mm_struct *mm = current->mm;
1348        
1349     *populate = 0;
1350 
1351     if (!len)
1352         return -EINVAL;
1353 
1354 #ifdef CONFIG_MSM_APP_SETTINGS
1355     if (use_app_setting)
1356         apply_app_setting_bit(file);
1357 #endif
1358 
1359     /*
1360      * Does the application expect PROT_READ to imply PROT_EXEC?
1361      *
1362      * (the exception is when the underlying filesystem is noexec
1363      *  mounted, in which case we dont add PROT_EXEC.)
1364      */
1365     if ((prot & PROT_READ) && (current->personality & READ_IMPLIES_EXEC))
1366         if (!(file && path_noexec(&file->f_path)))
1367             prot |= PROT_EXEC;
1368 
1369     if (!(flags & MAP_FIXED))
1370         addr = round_hint_to_min(addr);
1371 
1372     /* Careful about overflows.. */
1373     len = PAGE_ALIGN(len);
1374     if (!len)
1375         return -ENOMEM;
1376 
1377     /* offset overflow? */
1378     if ((pgoff + (len >> PAGE_SHIFT)) < pgoff)
1379         return -EOVERFLOW;
1380 
1381     /* Too many mappings? */
1382     if (mm->map_count > sysctl_max_map_count)
1383         return -ENOMEM;
1384 
1385     /* Obtain the address to map to. we verify (or select) it and ensure
1386      * that it represents a valid section of the address space.
1387      */
    	 // 找到一个合适大小的VMA区域的起始地址
1388     addr = get_unmapped_area(file, addr, len, pgoff, flags);
1389     if (offset_in_page(addr))
1390         return addr;
1391 
1392     /* Do simple checking here so the lower-level routines won't have
1393      * to. we assume access permissions have been handled by the open
1394      * of the memory object, so we don't do any here.
1395      */
    	 // 计算虚拟内存标记，calc_vm_prot_bits把用户态的PROT_开头的标记转换为VM_开头的标记
    	 // 函数calc_vm_flag_bits把MAP_开头标记转换为VM_开头的标记
    	 // mm->def_flags是默认的虚拟内存标记
1396     vm_flags |= calc_vm_prot_bits(prot) | calc_vm_flag_bits(flags) |
1397             mm->def_flags | VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC;
1398 
1399     if (flags & MAP_LOCKED)
1400         if (!can_do_mlock())
1401             return -EPERM;
1402 
1403     if (mlock_future_check(mm, vm_flags, len))
1404         return -EAGAIN;
1405 
1406     if (file) {
1407         struct inode *inode = file_inode(file);
1408 
1409         if (!file_mmap_ok(file, inode, pgoff, len))
1410             return -EOVERFLOW;
1411 
1412         switch (flags & MAP_TYPE) {
1413         case MAP_SHARED:
1414             if ((prot&PROT_WRITE) && !(file->f_mode&FMODE_WRITE))
1415                 return -EACCES;
1416 
1417             /*
1418              * Make sure we don't allow writing to an append-only
1419              * file..
1420              */
1421             if (IS_APPEND(inode) && (file->f_mode & FMODE_WRITE))
1422                 return -EACCES;
1423 
1424             /*
1425              * Make sure there are no mandatory locks on the file.
1426              */
1427             if (locks_verify_locked(file))
1428                 return -EAGAIN;
1429 
1430             vm_flags |= VM_SHARED | VM_MAYSHARE;
1431             if (!(file->f_mode & FMODE_WRITE))
1432                 vm_flags &= ~(VM_MAYWRITE | VM_SHARED);
1433 
1434             /* fall through */
1435         case MAP_PRIVATE:
1436             if (!(file->f_mode & FMODE_READ))
1437                 return -EACCES;
1438             if (path_noexec(&file->f_path)) {
1439                 if (vm_flags & VM_EXEC)
1440                     return -EPERM;
1441                 vm_flags &= ~VM_MAYEXEC;
1442             }
1443 
1444             if (!file->f_op->mmap)
1445                 return -ENODEV;
1446             if (vm_flags & (VM_GROWSDOWN|VM_GROWSUP))
1447                 return -EINVAL;
1448             break;
1449 
1450         default:
1451             return -EINVAL;
1452         }
1453     } else {
1454         switch (flags & MAP_TYPE) {
1455         case MAP_SHARED:
1456             if (vm_flags & (VM_GROWSDOWN|VM_GROWSUP))
1457                 return -EINVAL;
1458             /*
1459              * Ignore pgoff.
1460              */
1461             pgoff = 0;
1462             vm_flags |= VM_SHARED | VM_MAYSHARE;
1463             break;
1464         case MAP_PRIVATE:
1465             /*
1466              * Set pgoff according to addr for anon_vma.
1467              */
1468             pgoff = addr >> PAGE_SHIFT;
1469             break;
1470         default:
1471             return -EINVAL;
1472         }
1473     }
1474 
1475     /*
1476      * Set 'VM_NORESERVE' if we should not account for the
1477      * memory use of this mapping.
1478      */
1479     if (flags & MAP_NORESERVE) {
1480         /* We honor MAP_NORESERVE if allowed to overcommit */
1481         if (sysctl_overcommit_memory != OVERCOMMIT_NEVER)
1482             vm_flags |= VM_NORESERVE;
1483 
1484         /* hugetlb applies strict overcommit unless MAP_NORESERVE */
1485         if (file && is_file_hugepages(file))
1486             vm_flags |= VM_NORESERVE;
1487     }
1488 
    	 // 上面判断了文件页映射还是匿名页映射，并判断权限，设置VMA的访问权限
    	 // 根据上面找到的addr，创建VMA
1489     addr = mmap_region(file, addr, len, vm_flags, pgoff);
1490     if (!IS_ERR_VALUE(addr) &&
1491         ((vm_flags & VM_LOCKED) ||
1492          (flags & (MAP_POPULATE | MAP_NONBLOCK)) == MAP_POPULATE))
1493         *populate = len;
1494     return addr;
1495 }

2093 unsigned long
2094 get_unmapped_area(struct file *file, unsigned long addr, unsigned long len,
2095         unsigned long pgoff, unsigned long flags)
2096 {
2097     unsigned long (*get_area)(struct file *, unsigned long,
2098                   unsigned long, unsigned long, unsigned long);
2099 
2100     unsigned long error = arch_mmap_check(addr, len, flags);
2101     if (error)
2102         return error;
2103 
2104     /* Careful about overflows.. */
2105     if (len > TASK_SIZE)
2106         return -ENOMEM;
2107 
2108     get_area = current->mm->get_unmapped_area;
    	 // 如果是创建文件页映射或者巨型匿名页映射，调用文件系统给file的函数：file->f_op->get_unmapped_area，会将虚拟内存区域映射到文件，对虚拟内存区域的读写文件系统会转换为对文件的读写
2109     if (file && file->f_op->get_unmapped_area)
2110         get_area = file->f_op->get_unmapped_area;
    	 // 如果是匿名映射，直接调用current->mm->get_unmapped_area在进程的虚拟地址空间找到一个适合插入新VMA（addr,len）的地方，此时不需要映射文件
2111     addr = get_area(file, addr, len, pgoff, flags);
2112     if (IS_ERR_VALUE(addr))
2113         return addr;
2114 
2115     if (addr > TASK_SIZE - len)
2116         return -ENOMEM;
2117     if (offset_in_page(addr))
2118         return -EINVAL;
2119 
2120     addr = arch_rebalance_pgtables(addr, len);
2121     error = security_mmap_addr(addr);
2122     return error ? error : addr;
2123 }
2124 
2125 EXPORT_SYMBOL(get_unmapped_area);

// mm/mmap.c，真正的创建VMA并插入进程的VMA链表和红黑树
1624 unsigned long mmap_region(struct file *file, unsigned long addr,
1625         unsigned long len, vm_flags_t vm_flags, unsigned long pgoff)
1626 {
1627     struct mm_struct *mm = current->mm;
1628     struct vm_area_struct *vma, *prev;
1629     int error;
1630     struct rb_node **rb_link, *rb_parent;
1631     unsigned long charged = 0;
1632 
1633     /* Check against address space limit. */
    	 // 检查进程分配的虚拟内存是否超过限制
1634     if (!may_expand_vm(mm, len >> PAGE_SHIFT)) {
1635         unsigned long nr_pages;
1636 
1637         /*
1638          * MAP_FIXED may remove pages of mappings that intersects with
1639          * requested mapping. Account for the pages it would unmap.
1640          */
1641         if (!(vm_flags & MAP_FIXED))
1642             return -ENOMEM;
1643 
1644         nr_pages = count_vma_pages_range(mm, addr, addr + len);
1645 
1646         if (!may_expand_vm(mm, (len >> PAGE_SHIFT) - nr_pages))
1647             return -ENOMEM;
1648     }
1649 
1650     /* Clear old maps */
1651     while (find_vma_links(mm, addr, addr + len, &prev, &rb_link,
1652                   &rb_parent)) {
1653         if (do_munmap(mm, addr, len))
1654             return -ENOMEM;
1655     }
1656 
1657     /*
1658      * Private writable mapping: check memory availability
1659      */
1660     if (accountable_mapping(file, vm_flags)) {
1661         charged = len >> PAGE_SHIFT;
1662         if (security_vm_enough_memory_mm(mm, charged))
1663             return -ENOMEM;
1664         vm_flags |= VM_ACCOUNT;
1665     }
1666 
1667     /*
1668      * Can we just expand an old mapping?
1669      */
1670     vma = vma_merge(mm, prev, addr, addr + len, vm_flags,
1671             NULL, file, pgoff, NULL, NULL_VM_UFFD_CTX, NULL);
1672     if (vma)
1673         goto out;
1674 
1675     /*
1676      * Determine the object being mapped and call the appropriate
1677      * specific mapper. the address has already been validated, but
1678      * not unmapped, but the maps are removed from the list.
1679      */
    	 // 不能合并，创建新的vma，从vm_area_cachep的slub缓存中
1680     vma = kmem_cache_zalloc(vm_area_cachep, GFP_KERNEL);
1681     if (!vma) {
1682         error = -ENOMEM;
1683         goto unacct_error;
1684     }
1685 
1686     vma->vm_mm = mm;
1687     vma->vm_start = addr;
1688     vma->vm_end = addr + len;
1689     vma->vm_flags = vm_flags;
1690     vma->vm_page_prot = vm_get_page_prot(vm_flags);
1691     vma->vm_pgoff = pgoff;
1692     INIT_LIST_HEAD(&vma->anon_vma_chain);
1693 	
    	 // 文件映射
1694     if (file) {
1695         if (vm_flags & VM_DENYWRITE) {
1696             error = deny_write_access(file);
1697             if (error)
1698                 goto free_vma;
1699         }
1700         if (vm_flags & VM_SHARED) {
1701             error = mapping_map_writable(file->f_mapping);
1702             if (error)
1703                 goto allow_write_and_free_vma;
1704         }
1705 
1706         /* ->mmap() can change vma->vm_file, but must guarantee that
1707          * vma_link() below can deny write-access if VM_DENYWRITE is set
1708          * and map writably if VM_SHARED is set. This usually means the
1709          * new file must not have been exposed to user-space, yet.
1710          */
1711         vma->vm_file = get_file(file);
    		 // 不同的文件系统和驱动可以实现自己的映射方法，Linux一切皆文件
1712         error = file->f_op->mmap(file, vma);
1713         if (error)
1714             goto unmap_and_free_vma;
1715 
1716         /* Can addr have changed??
1717          *
1718          * Answer: Yes, several device drivers can do it in their
1719          *         f_op->mmap method. -DaveM
1720          * Bug: If addr is changed, prev, rb_link, rb_parent should
1721          *      be updated for vma_link()
1722          */
1723         WARN_ON_ONCE(addr != vma->vm_start);
1724 
1725         addr = vma->vm_start;
1726         vm_flags = vma->vm_flags;
1727     } else if (vm_flags & VM_SHARED) {
1728         error = shmem_zero_setup(vma);
1729         if (error)
1730             goto free_vma;
1731     }
1732 
    	 // 把VMA插入链表和红黑树
1733     vma_link(mm, vma, prev, rb_link, rb_parent);
1734     /* Once vma denies write, undo our temporary denial count */
1735     if (file) {
1736         if (vm_flags & VM_SHARED)
1737             mapping_unmap_writable(file->f_mapping);
1738         if (vm_flags & VM_DENYWRITE)
1739             allow_write_access(file);
1740     }
1741     file = vma->vm_file;
1742 out:
1743     perf_event_mmap(vma);
1744 
1745     vm_stat_account(mm, vm_flags, file, len >> PAGE_SHIFT);
1746     if (vm_flags & VM_LOCKED) {
1747         if (!((vm_flags & VM_SPECIAL) || is_vm_hugetlb_page(vma) ||
1748                     vma == get_gate_vma(current->mm)))
1749             mm->locked_vm += (len >> PAGE_SHIFT);
1750         else
1751             vma->vm_flags &= VM_LOCKED_CLEAR_MASK;
1752     }
1753 
1754     if (file)
1755         uprobe_mmap(vma);
1756 
1757     /*
1758      * New (or expanded) vma always get soft dirty status.
1759      * Otherwise user-space soft-dirty page tracker won't
1760      * be able to distinguish situation when vma area unmapped,
1761      * then new mapped in-place (which must be aimed as
1762      * a completely new data area).
1763      */
1764     vma->vm_flags |= VM_SOFTDIRTY;
1765 
1766     vma_set_page_prot(vma);
1767 
1768     return addr;
1769 
1770 unmap_and_free_vma:
1771     vma->vm_file = NULL;
1772     fput(file);
1773 
1774     /* Undo any partial mapping done by a device driver. */
1775     unmap_region(mm, vma, prev, vma->vm_start, vma->vm_end);
1776     charged = 0;
1777     if (vm_flags & VM_SHARED)
1778         mapping_unmap_writable(file->f_mapping);
1779 allow_write_and_free_vma:
1780     if (vm_flags & VM_DENYWRITE)
1781         allow_write_access(file);
1782 free_vma:
1783     kmem_cache_free(vm_area_cachep, vma);
1784 unacct_error:
1785     if (charged)
1786         vm_unacct_memory(charged);
1787     return error;
1788 }
```

