```
https://blog.csdn.net/yusakul/article/details/105706674
https://blog.csdn.net/weixin_42523774/article/details/103341058（glibc）
https://blog.csdn.net/yhb1047818384/article/details/51842607（寄存器）
https://www.cnblogs.com/nufangrensheng/p/3890856.html（::memory）
https://baijiahao.baidu.com/s?id=1604601045858159778&wfr=spider&for=pc（系统调用实现文件）
```

### 1、write流程

1、user调用write系统调用

2、glibc中转换为系统调用号，将系统调用号保存在x8寄存器中，同时保存返回地址在x0，接着触发svc异常

3、从EL0切换到EL1，进入内核通过aarch64的中断向量表，找到异常向量入口el0_sync（arch/arm64/kernel/entry.S）

4、el0_sync函数中判断异常类型是svc触发的异常

```
lsr x24, x25, #ESR_ELx_EC_SHIFT // exception class
```

5、接着进入el0_svc

```
b.eq    el0_svc
```

6、加载异常系统调用表

```
adrp    stbl, sys_call_table        // load syscall table pointer
uxtw    scno, w8            // syscall number in w8，从x8寄存起低32位中找打系统调用号
ldr x16, [stbl, scno, lsl #3]   // address in the syscall table
blr x16             // call sys_* routine
b   ret_fast_syscall		// 系统调用返回
```

7、第六步找通过系统调用号，在系统调用表中找到write的系统调用的位置

```
include/uapi/asm-generic/unistd.h	
    /* fs/read_write.c */
    #define __NR_write 64

fs/read_write.c
	SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,size_t, count)
```

8、写入操作

```
 577 SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
 578         size_t, count)
 579 {
 580     struct fd f = fdget_pos(fd);
 581     ssize_t ret = -EBADF;
 582 
 583     if (f.file) {
 584         loff_t pos = file_pos_read(f.file);
 585         ret = vfs_write(f.file, buf, count, &pos);
 586         if (ret >= 0)
 587             file_pos_write(f.file, pos);
 588         fdput_pos(f);		// 写入
 589     }
 590 
 591     return ret;
 592 }
```

9、vfs_write内部调用__vfs_write完成用户buf到iovec的映射

```
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
    ssize_t ret; 

    if (!(file->f_mode & FMODE_WRITE))
        return -EBADF;
    if (!(file->f_mode & FMODE_CAN_WRITE))
        return -EINVAL;
    if (unlikely(!access_ok(VERIFY_READ, buf, count)))
        return -EFAULT;

    ret = rw_verify_area(WRITE, file, pos, count);
    if (ret >= 0) { 
        count = ret; 
        file_start_write(file);
        ret = __vfs_write(file, buf, count, pos);
        if (ret > 0) { 
            fsnotify_modify(file);
            add_wchar(current, ret);
        }    
        inc_syscw(current);
        file_end_write(file);
    }    

    return ret; 
}

ssize_t __vfs_write(struct file *file, const char __user *p, size_t count,
            loff_t *pos)
{       
    if (file->f_op->write)
        return file->f_op->write(file, p, count, pos);
    else if (file->f_op->write_iter)
    	// 对于块设备文件，由于file_operations中没有实现write,所以会走else分支
        return new_sync_write(file, p, count, pos);
    else
        return -EINVAL;
}

static ssize_t new_sync_write(struct file *filp, const char __user *buf, size_t len, loff_t *ppos)
{
	// 用户空间的buf映射到iovec结构体中，这是临时变量
    struct iovec iov = { .iov_base = (void __user *)buf, .iov_len = len };
    struct kiocb kiocb;
    struct iov_iter iter;
    ssize_t ret;
    
    // 初始化kernel的io控制块
    init_sync_kiocb(&kiocb, filp);
    kiocb.ki_pos = *ppos;
    // 初始化 iter（struct iov_iter）
    iov_iter_init(&iter, WRITE, &iov, 1, len);
    
    // 这里是介绍直接写块设备文件，所以文件系统是bdevfs,对应文件(fs/block_dev.c)
    // 该文件中实现了file_operations中的write_iter就是下面要介绍的写函数
    ret = filp->f_op->write_iter(&kiocb, &iter);
    BUG_ON(ret == -EIOCBQUEUED);
    if (ret > 0)
        *ppos = kiocb.ki_pos;
    return ret;
}

// 参考：https://www.cnblogs.com/hanyan225/archive/2010/10/24/1859655.html
/*
	每一个io请求在内核对应一个kiocb
*/
struct kiocb {
    struct file     *ki_filp;   // 指向打开文件对应的file结构体
    loff_t          ki_pos;   // 当前写文件的偏移
    void (*ki_complete)(struct kiocb *iocb, long ret, long ret2);
    void            *private; // 可以被文件系统层自由使用
    int         ki_flags;   // open打开文件时的flag
};

struct iov_iter {
    int type;				// 读或者写
    size_t iov_offset;		// 初始化为0
    size_t count;			// 用户buf的总长度
    union {
        const struct iovec *iov;
        const struct kvec *kvec;
        const struct bio_vec *bvec;
    };
    unsigned long nr_segs;	// 上面数组中成员个数，这里是1
};
```

10、__vfs_write调用具体文件系统的write_iter，这里介绍直接打开块设备文件写过程，块设备文件对应块设备文件系统bdevfs（伪文件系统，不会mount，只供内核使用）

```
ssize_t blkdev_write_iter(struct kiocb *iocb, struct iov_iter *from)
{  
	// 下面的file就是kernel io控制块对应的file结构
    struct file *file = iocb->ki_filp;
    // inode是块设备文件对应的主inode，file->f_mapping是struct address_space类型，它的host代表
    // 该地址空间属于哪个文件
    struct inode *bd_inode = file->f_mapping->host;
    // 获取文件长度，inode->i_size
    loff_t size = i_size_read(bd_inode);
    // 对应进程的plug
    struct blk_plug plug;
    ssize_t ret;

    if (bdev_read_only(I_BDEV(bd_inode)))
        return -EPERM;
    
    // 判断用户写的长度是否为0
    if (!iov_iter_count(from))
        return 0;

	// 判断写的位置是否超过文件大小
    if (iocb->ki_pos >= size)
        return -ENOSPC;

    iov_iter_truncate(from, size - iocb->ki_pos);

	// 初始化进程的plug，task_struct->plug
    blk_start_plug(&plug);
    // 开始写
    ret = __generic_file_write_iter(iocb, from);
    if (ret > 0) {
    	// 写了多少字节，更新file结构的pos
        ssize_t err;
        err = generic_write_sync(file, iocb->ki_pos - ret, ret);
        if (err < 0)
            ret = err;
    }
    // 进行unplug
    blk_finish_plug(&plug);
    return ret;
}
```



### 2、系统调用表

```
arch/arm64/kernel/sys.c
    void * const sys_call_table[__NR_syscalls] __aligned(4096) = {
    	[0 ... __NR_syscalls - 1] = sys_ni_syscall,	// 没有实现的系统调用会跳转到这里，内部直接返回
    #include <asm/unistd.h>	 // arch/arm64/include/asm/unistd.h
    };

arch/arm64/include/asm/unistd.h:56
	#define NR_syscalls (__NR_syscalls)
	
arch/arm64/include/uapi/asm/unistd.h
	#include <asm-generic/unistd.h>

include/asm-generic/unistd.h
	#include <uapi/asm-generic/unistd.h>
	
include/uapi/asm-generic/unistd.h	
	该文件中说明了系统调用的实现的文件
		write系统调用在fs/read_write.c  //SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,size_t, count)
		getpid在kernel/sys.c // SYSCALL_DEFINE0(getpid)
	#define __NR_syscalls 285  // 当前系统有285个系统调用
	
include/linux/syscalls.h
	对系统调用申明
```

### 3、系统调用write的原子性讨论

https://blog.csdn.net/dog250/article/details/78879600

append模式通过锁主inode来保证原子，write系统调用本身并步保证原子

###  4、基于EXT4的写

与上面基于裸设备的写IO不同，EXT4实现了自己的file_operations(drivers/fs/ext4/file.c)

承接上面的__vfs_write调用具体的文件系统的写操作函数，基于ext4的写操作的流程如下

ext4_file_write_iter->__generic_file_write_iter->generic_file_direct_write->filemap_write_and_wait_range->__filemap_fdatawrite_range->do_writepages->generic_writepages->write_cache_pages->__writepage->ext4_writepage->ext4_bio_write_page->io_submit_add_bh

参考：

https://www.cnblogs.com/f-ck-need-u/p/7016077.html（阿里云团队）

https://blog.csdn.net/qq_32473685/article/details/103494398

1、fs/ext4/file.c中定义了ext4文件系统的file_operations中ext4_file_write_iter实现了ext4提供给vfs的写文件操作

```
static ssize_t
ext4_file_write_iter(struct kiocb *iocb, struct iov_iter *from)
{
    struct file *file = iocb->ki_filp;
    struct inode *inode = file_inode(iocb->ki_filp);
    struct mutex *aio_mutex = NULL;
    struct blk_plug plug;
    // 这里将内核io控制块设置为直接写，决定了__generic_file_write_iter函数内部的分支判断
    int o_direct = iocb->ki_flags & IOCB_DIRECT;
    int overwrite = 0;
    ssize_t ret;

    /*
     * Unaligned direct AIO must be serialized; see comment above
     * In the case of O_APPEND, assume that we must always serialize
     */
    if (o_direct &&
        ext4_test_inode_flag(inode, EXT4_INODE_EXTENTS) &&
        !is_sync_kiocb(iocb) &&
        (iocb->ki_flags & IOCB_APPEND ||
         ext4_unaligned_aio(inode, from, iocb->ki_pos))) {
        aio_mutex = ext4_aio_mutex(inode);
        mutex_lock(aio_mutex);
        ext4_unwritten_wait(inode);
    }

    mutex_lock(&inode->i_mutex);
    ret = generic_write_checks(iocb, from);
    if (ret <= 0)
        goto out;

    /*
     * If we have encountered a bitmap-format file, the size limit
     * is smaller than s_maxbytes, which is for extent-mapped files.
     */
    if (!(ext4_test_inode_flag(inode, EXT4_INODE_EXTENTS))) {
        struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);

        if (iocb->ki_pos >= sbi->s_bitmap_maxbytes) {
            ret = -EFBIG;
            goto out;
        }
        iov_iter_truncate(from, sbi->s_bitmap_maxbytes - iocb->ki_pos);
    }

    iocb->private = &overwrite;
    if (o_direct) {
        size_t length = iov_iter_count(from);
        loff_t pos = iocb->ki_pos;
        blk_start_plug(&plug);

        /* check whether we do a DIO overwrite or not */
        if (ext4_should_dioread_nolock(inode) && !aio_mutex &&
            !file->f_mapping->nrpages && pos + length <= i_size_read(inode)) {
            struct ext4_map_blocks map;
            unsigned int blkbits = inode->i_blkbits;
            int err, len;

            map.m_lblk = pos >> blkbits;
            map.m_len = (EXT4_BLOCK_ALIGN(pos + length, blkbits) >> blkbits)
                - map.m_lblk;
            len = map.m_len;

            err = ext4_map_blocks(NULL, inode, &map, 0);
            /*
             * 'err==len' means that all of blocks has
             * been preallocated no matter they are
             * initialized or not.  For excluding
             * unwritten extents, we need to check
             * m_flags.  There are two conditions that
             * indicate for initialized extents.  1) If we
             * hit extent cache, EXT4_MAP_MAPPED flag is
             * returned; 2) If we do a real lookup,
             * non-flags are returned.  So we should check
             * these two conditions.
             */
            if (err == len && (map.m_flags & EXT4_MAP_MAPPED))
                overwrite = 1;
        }
    }

	/*
		参数：
			iocb：内核IO控制块
			from：iov_iter结构的io向量
		说明：下面函数执行写入from指向的数据io向量
	*/
    ret = __generic_file_write_iter(iocb, from);
    mutex_unlock(&inode->i_mutex);

    if (ret > 0) {
        ssize_t err;

        err = generic_write_sync(file, iocb->ki_pos - ret, ret);
        if (err < 0)
            ret = err;
    }
    if (o_direct)
        blk_finish_plug(&plug);

    if (aio_mutex)
        mutex_unlock(aio_mutex);
    return ret;

out:
    mutex_unlock(&inode->i_mutex);
    if (aio_mutex)
        mutex_unlock(aio_mutex);
    return ret;
}
```

2、generic_file_direct_write

```
ssize_t
generic_file_direct_write(struct kiocb *iocb, struct iov_iter *from, loff_t pos)
{
    struct file *file = iocb->ki_filp;
    struct address_space *mapping = file->f_mapping;
    struct inode    *inode = mapping->host;
    ssize_t     written;
    size_t      write_len;
    pgoff_t     end;
    struct iov_iter data;
	
	// 计算总的要写入的数据的长度
    write_len = iov_iter_count(from);
    // 结尾位置要进行页对齐
    end = (pos + write_len - 1) >> PAGE_CACHE_SHIFT;

    written = filemap_write_and_wait_range(mapping, pos, pos + write_len - 1);
    if (written)
        goto out;

    /*
     * After a write we want buffered reads to be sure to go to disk to get
     * the new data.  We invalidate clean cached page from the region we're
     * about to write.  We do this *before* the write so that we can return
     * without clobbering -EIOCBQUEUED from ->direct_IO().
     */
    if (mapping->nrpages) {
        written = invalidate_inode_pages2_range(mapping,
                    pos >> PAGE_CACHE_SHIFT, end);
        /*
         * If a page can not be invalidated, return 0 to fall back
         * to buffered write.
         */
        if (written) {
            if (written == -EBUSY)
                return 0;
            goto out;
        }
    }
    
    data = *from;
    written = mapping->a_ops->direct_IO(iocb, &data, pos);

    /*
     * Finally, try again to invalidate clean pages which might have been
     * cached by non-direct readahead, or faulted in by get_user_pages()
     * if the source of the write was an mmap'ed region of the file
     * we're writing.  Either one is a pretty crazy thing to do,
     * so we don't support it 100%.  If this invalidation
     * fails, tough, the write still worked...
     */
    if (mapping->nrpages) {
        invalidate_inode_pages2_range(mapping,
                          pos >> PAGE_CACHE_SHIFT, end);
    }

    if (written > 0) {
        pos += written;
        iov_iter_advance(from, written);
        if (pos > i_size_read(inode) && !S_ISBLK(inode->i_mode)) {
            i_size_write(inode, pos);
            mark_inode_dirty(inode);
        }
        iocb->ki_pos = pos;
    }
out:
    return written;
}
```

3、经过各种函数最终到达ext的address_space_operations的ext4_writepage

```
file：drivers/fs/ext4/inode.c

```


