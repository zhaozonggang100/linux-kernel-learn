### 1、流程

1、从应用层到内核通过系统调用产生的软件中断在系统调用表找到对应的系统调用，和write类似
```
--------> include/linux/file.h
29 struct fd {
30     struct file *file;
31     unsigned int flags;
32 };

-------->fs/read_write.c
562 SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
563 {   
564     struct fd f = fdget_pos(fd);	// 从当前进程的task_struct中取file对象，增加file引用计数
565     ssize_t ret = -EBADF;
566     
567     if (f.file) {
568         loff_t pos = file_pos_read(f.file); // 获取当前读写位置
569         ret = vfs_read(f.file, buf, count, &pos); // 真正开始读
570         if (ret >= 0)
571             file_pos_write(f.file, pos);  // 读取成功返回当前的文件偏移
572         fdput_pos(f);	// 更新文件引用计数，减少引用计数，如果为0，释放file
573     }
574     return ret;    // 返回读取到的字节数
575 }
```

2、使用vfs_read传递读的过程，以便后续通过文件的inode找到文件对应的文件系统  

```
-------->fs/read_write.c
440 ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
441 {   
442     ssize_t ret;
443     
444     if (!(file->f_mode & FMODE_READ))  // 检查文件处于可读模式
445         return -EBADF;
446     if (!(file->f_mode & FMODE_CAN_READ))
447         return -EINVAL;
448     if (unlikely(!access_ok(VERIFY_WRITE, buf, count)))  // 检查用户buf可访问
449         return -EFAULT;
450     
451     ret = rw_verify_area(READ, file, pos, count); // 检测要访问的文件是否有锁冲突
452     if (ret >= 0) {
453         count = ret;
454         ret = __vfs_read(file, buf, count, pos);
455         if (ret > 0) {
456             fsnotify_access(file);
457             add_rchar(current, ret);
458         }
459         inc_syscr(current);
460     }   
461 
462     return ret;
463 }

如果文件操作表中定义了read回调方法，则调用之。否则调用系统默认实现new_sync_read函数
428 ssize_t __vfs_read(struct file *file, char __user *buf, size_t count,
429            loff_t *pos)
430 {
431     if (file->f_op->read)
432         return file->f_op->read(file, buf, count, pos);
433     else if (file->f_op->read_iter)
434         return new_sync_read(file, buf, count, pos); // 块设备文件走这一步
435     else
436         return -EINVAL;
437 }

struct file {
	const struct file_operations    *f_op;
}

/*
	定义：include/uapi/linux/uio.h
	说明：用户read对应的用户空间的缓存区，也就是关联到进程的VMA结构(红黑树)
*/
16 struct iovec { 
		// 用户空间的缓存区地址
18     void __user *iov_base;  /* BSD uses caddr_t (1003.1g requires void *) */
		// 用户空间的缓存区长度
19     __kernel_size_t iov_len; /* Must be size_t (1003.1g) */
20 };
/*
	定义：include/linux/fs.h
	说明：内核为了支持用户空间的AIO实现的io控制块
*/
324 struct kiocb { 
325     struct file     *ki_filp;
326     loff_t          ki_pos;
327     void (*ki_complete)(struct kiocb *iocb, long ret, long ret2);
328     void            *private;
329     int         ki_flags;
330 };
/*
	定义：include/linux/uio.h
	说明：
*/
28 struct iov_iter {
29     int type;
30     size_t iov_offset;
31     size_t count;
32     union {
33         const struct iovec *iov;
34         const struct kvec *kvec;
35         const struct bio_vec *bvec;
36     };
37     unsigned long nr_segs;
38 };
/*
	定义：块设备文件的读（fs/read_write.c）
	说明：用户空间的IO读取，相当于是针对IO向量数组的，一个IO向量被看作是一个IO请求段
	readv系统调用需要构造iovec的数组，read只需要构造一个长度为1的向量数组
*/
411 static ssize_t new_sync_read(struct file *filp, char __user *buf, size_t len, loff_t *ppos)
412 {
		// 代表用户空间的缓存区向量
413     struct iovec iov = { .iov_base = buf, .iov_len = len };
414     struct kiocb kiocb;
415     struct iov_iter iter;
416     ssize_t ret;
417 
418     init_sync_kiocb(&kiocb, filp);
419     kiocb.ki_pos = *ppos;
420     iov_iter_init(&iter, READ, &iov, 1, len);
421 
		// 下面调用了第三步中块设备文件file_operations中的read_iter函数即		
		// blkdev_read_iter函数
422     ret = filp->f_op->read_iter(&kiocb, &iter);
423     BUG_ON(ret == -EIOCBQUEUED);
424     *ppos = kiocb.ki_pos;
425     return ret;
426 }
```

3、根据文件系统和读写方式的不同vfs_read会调用不同文件系统的read实现

```
1、这里介绍绕过普通文件系统直接读块设备，基于磁盘的文件读写，不管是否绕过普通文件系统（ext4等），在通用块设备层都要经过bdevfs（块设备文件系统）找到块设备文件，接着根据块设备文件得到块设备对应的block_device结构体，该结构体中有一个struct gendisk的成员，这里简单说一下block_device和gendisk的关系，block_device代表一个块设备或者是块设备上的一个分区，而gendisk是整个块设备在通用块设备层的抽象（不管一个块设备有多少分区，一个块设备就一个）

对于gendisk和block_device的详解参考：
https://blog.csdn.net/vanbreaker/article/details/8265706

普通文件系统之下的块设备实现块设备对应的buffer cache（地址空间）的操作函数集，具体如下：
fs/block_dev.c
static const struct address_space_operations def_blk_aops = {
    .readpage   = blkdev_readpage,
    .readpages  = blkdev_readpages,
    .writepage  = blkdev_writepage,
    .write_begin    = blkdev_write_begin,
    .write_end  = blkdev_write_end,
    .writepages = generic_writepages,
    .releasepage    = blkdev_releasepage,
    .direct_IO  = blkdev_direct_IO,
    .is_dirty_writeback = buffer_check_dirty_writeback,
};

块设备提供给上层文件系统(vfs或者ext4)的操作函数集是，：
fs/block_dev.c
// 字符设备注册到vfs也会提供对应的file_operations
const struct file_operations def_blk_fops = {
    .open       = blkdev_open,    // open("/dev/sdx")时vfs会调用
    .release    = blkdev_close,
    .llseek     = block_llseek,
    .read_iter  = blkdev_read_iter, // 块设备文件只注册了该函数，所以第二步的vfs_read中会调用new_sync_read
    .write_iter = blkdev_write_iter,
    .mmap       = generic_file_mmap,
    .fsync      = blkdev_fsync,
    .unlocked_ioctl = block_ioctl,
#ifdef CONFIG_COMPAT
    .compat_ioctl   = compat_blkdev_ioctl,
#endif
    .splice_read    = generic_file_splice_read,
    .splice_write   = iter_file_splice_write,
};

块设备文件的inode中的成员umode_t i_mode代表该成员是否对应一个块设备

/*
	定义：fs/block_dev.c
	说明：块设备文件对应到vfs_read调用的读取函数，相当于vfs直接绕过普通文件系统（ext4）
		直接调用bdevfs中文件对应的file_operations中的读函数
*/
ssize_t blkdev_read_iter(struct kiocb *iocb, struct iov_iter *to)
{   
    struct file *file = iocb->ki_filp;
    struct inode *bd_inode = file->f_mapping->host;
    loff_t size = i_size_read(bd_inode);
    loff_t pos = iocb->ki_pos;

    if (pos >= size)
        return 0; 
    
    size -= pos;
    iov_iter_truncate(to, size);
    
    // 所有文件系统的读实现都要调用下面的读函数
    // 下面的函数最终会调用do_generic_file_read进行读
    return generic_file_read_iter(iocb, to);
}
```

4、最终的核心在generic_file_read_iter

```
参考：
https://blog.csdn.net/frank_zyp/article/details/88853932（针对普通文件系统）
https://blog.csdn.net/weixin_36145588/article/details/74937592（块设备文件系统）

static ssize_t do_generic_file_read(struct file *filp, loff_t *ppos,
        struct iov_iter *iter, ssize_t written)
{
	struct address_space *mapping = filp->f_mapping;
    struct inode *inode = mapping->host;
    struct file_ra_state *ra = &filp->f_ra;
    pgoff_t index;
    pgoff_t last_index;
    pgoff_t prev_index;
    unsigned long offset;      /* offset into pagecache page */
    unsigned int prev_offset;
    int error = 0;
    
    index = *ppos >> PAGE_CACHE_SHIFT;
    prev_index = ra->prev_pos >> PAGE_CACHE_SHIFT;
    prev_offset = ra->prev_pos & (PAGE_CACHE_SIZE-1);
    last_index = (*ppos + iter->count + PAGE_CACHE_SIZE-1) >> PAGE_CACHE_SHIFT;
    offset = *ppos & ~PAGE_CACHE_MASK;
    
    for (;;) {
        struct page *page;
        pgoff_t end_index;
        loff_t isize;
        unsigned long nr, ret;

        cond_resched();
find_page:
        if (fatal_signal_pending(current)) {
            error = -EINTR;
            goto out;
        }
		
		// 这里会通过open是file结构对应的f_mapping（地址空间）以及页偏移
		// 看看是否有对应的页缓存，在这里可以判断文件的读取缓存是否命中
		// （file结构中有path代表进程当前打开的文件全路径）
        page = find_get_page(mapping, index);
        if (!page) {
        	// 缓存未命中，要进行预读
            page_cache_sync_readahead(mapping,
                    ra, filp,
                    index, last_index - index);
            page = find_get_page(mapping, index);
            if (unlikely(page == NULL))
            	// 此处预读还是没有对应page，就需要创建page并建立和磁盘数据的映射
                goto no_cached_page;
        }
        if (PageReadahead(page)) {
            page_cache_async_readahead(mapping,
                    ra, filp, page,
                    index, last_index - index);
        }
        if (!PageUptodate(page)) {
            /*
             * See comment in do_read_cache_page on why
             * wait_on_page_locked is used to avoid unnecessarily
             * serialisations and why it's safe.
             */
            wait_on_page_locked_killable(page);
            if (PageUptodate(page))
                goto page_ok;

            if (inode->i_blkbits == PAGE_CACHE_SHIFT ||
                    !mapping->a_ops->is_partially_uptodate)
                goto page_not_up_to_date;
            if (!trylock_page(page))
                goto page_not_up_to_date;
            /* Did it get truncated before we got the lock? */
            if (!page->mapping)
                goto page_not_up_to_date_locked;
            if (!mapping->a_ops->is_partially_uptodate(page,
                            offset, iter->count))
                goto page_not_up_to_date_locked;
            unlock_page(page);
        }
page_ok:
        /*
             * i_size must be checked after we know the page is Uptodate.
             *
             * Checking i_size after the check allows us to calculate
             * the correct value for "nr", which means the zero-filled
             * part of the page is not copied back to userspace (unless
             * another truncate extends the file - this is desired though).
             */

            isize = i_size_read(inode);
            end_index = (isize - 1) >> PAGE_CACHE_SHIFT;
            if (unlikely(!isize || index > end_index)) {
                page_cache_release(page);
                goto out;
            }

            /* nr is the maximum number of bytes to copy from this page */
            nr = PAGE_CACHE_SIZE;
            if (index == end_index) {
                nr = ((isize - 1) & ~PAGE_CACHE_MASK) + 1;
                if (nr <= offset) {
                    page_cache_release(page);
                    goto out;
                }
            }
            nr = nr - offset;

            /* If users can be writing to this page using arbitrary
             * virtual addresses, take care about potential aliasing
             * before reading the page on the kernel side.
             */
            if (mapping_writably_mapped(mapping))
                flush_dcache_page(page);

            /*
             * When a sequential read accesses a page several times,
             * only mark it as accessed the first time.
             */
            if (prev_index != index || offset != prev_offset)
                mark_page_accessed(page);
            prev_index = index;

            /*
             * Ok, we have the page, and it's up-to-date, so
             * now we can copy it to user space...
             */

            ret = copy_page_to_iter(page, offset, nr, iter);
            offset += ret;
            index += offset >> PAGE_CACHE_SHIFT;
            offset &= ~PAGE_CACHE_MASK;
            prev_offset = offset;

            page_cache_release(page);
            written += ret;
            if (!iov_iter_count(iter))
            goto out;
            if (ret < nr) {
                error = -EFAULT;
                goto out;
            }
            continue;

page_not_up_to_date:
            /* Get exclusive access to the page ... */
            error = lock_page_killable(page);
            if (unlikely(error))
                goto readpage_error;

page_not_up_to_date_locked:
            /* Did it get truncated before we got the lock? */
            if (!page->mapping) {
                unlock_page(page);
                page_cache_release(page);
                continue;
            }
            /* Did somebody else fill it already? */
            if (PageUptodate(page)) {
                unlock_page(page);
                goto page_ok;
            }

readpage:
			/*
				缓存未命中就需要从磁盘读取数据并填充alloc的page
				下面的mapping->a_ops->readpage对应块设备的
				static const struct address_space_operations def_blk_aops
				中的blkdev_readpage成员（.readpage）
			*/
            /*
             * A previous I/O error may have been due to temporary
             * failures, eg. multipath errors.
             * PG_error will be set again if readpage fails.
             */
            ClearPageError(page);
            /* Start the actual read. The read will unlock the page. */
            error = mapping->a_ops->readpage(filp, page);

            if (unlikely(error)) {
                if (error == AOP_TRUNCATED_PAGE) {
                    page_cache_release(page);
                    error = 0;
                    goto find_page;
                }
                goto readpage_error;
            }

            if (!PageUptodate(page)) {
                error = lock_page_killable(page);
                if (unlikely(error))
                    goto readpage_error;
                if (!PageUptodate(page)) {
                    if (page->mapping == NULL) {
                        /*
                         * invalidate_mapping_pages got it
                         */
                        unlock_page(page);
                        page_cache_release(page);
                        goto find_page;
                    }
                    unlock_page(page);
                    shrink_readahead_size_eio(filp, ra);
                    error = -EIO;
                    goto readpage_error;
                }
                unlock_page(page);
            }

            goto page_ok;

readpage_error:
        /* UHHUH! A synchronous read error occurred. Report it */
        page_cache_release(page);
        goto out;

no_cached_page:
		/*
         * Ok, it wasn't cached, so we need to create a new
         * page..
         */
        page = page_cache_alloc_cold(mapping);
        if (!page) {
            error = -ENOMEM;
            goto out;
        }
        error = add_to_page_cache_lru(page, mapping, index,
                mapping_gfp_constraint(mapping, GFP_KERNEL));
        if (error) {
            page_cache_release(page);
            if (error == -EEXIST) {
                error = 0;
                goto find_page;
            }
            goto out;
        }
        goto readpage;
    }
out:
    ra->prev_pos = prev_index;
    ra->prev_pos <<= PAGE_CACHE_SHIFT;
    ra->prev_pos |= prev_offset;

    *ppos = ((loff_t)index << PAGE_CACHE_SHIFT) + offset;
    file_accessed(filp);
    return written ? written : error;
}
```

5、上一步读的时候如果数据不在buffer cache中，就要创建page并加入块设备inode对应的address_space中，然后向bio层发起io请求，bio主要负责将page映射到块设备的block，并通过块设备驱动进行磁盘数据读取，读取的过程一般会先将page转换成bio请求，再将bio请求加入块设备gendisk的request_queue中，加入的过程可能会涉及到bio的合并，会将bio映射的地址连续的请求合并成一个request加入到request_queue，接着IO调度（CFQ）器会进行相应的request排序，然后IO调度器在合适的时机唤醒块设备驱动绑定的request处理内核线程，线程会从IO调度器也就是gendisk中已经被排序的request_queue中fetch一个request，接着向块设备发起DMA请求进行数据读取，读完数据在request中，最后会将request中的数据映射到读请求对应的page中，然后映射到用户地址空间（VMA）中，一次完整的读请求结束（当然这里分析的同步读取，感兴趣的伙伴可以研究下异步读取过程，应该就是在读到数据调用用户注册的回调函数）

```
static int blkdev_readpage(struct file * file, struct page * page)
{
    return block_read_full_page(page, blkdev_get_block);
}

/*
	定义：fs/buffer.c
	参考：https://blog.csdn.net/weixin_36145588/article/details/74937592
*/
int block_read_full_page(struct page *page, get_block_t *get_block)
{
    struct inode *inode = page->mapping->host;
    sector_t iblock, lblock;
    // buffer_head就是buffer cache对应的page的链表头
    struct buffer_head *bh, *head, *arr[MAX_BUF_PER_PAGE];
    unsigned int blocksize, bbits;
    int nr, i;
    int fully_mapped = 1; 

    head = create_page_buffers(page, inode, 0);
    blocksize = head->b_size;
    bbits = block_size_bits(blocksize);

    iblock = (sector_t)page->index << (PAGE_CACHE_SHIFT - bbits);
    lblock = (i_size_read(inode)+blocksize-1) >> bbits;
    bh = head;
    nr = 0; 
    i = 0; 

    do { 
        if (buffer_uptodate(bh))
            continue;

        if (!buffer_mapped(bh)) {
            int err = 0; 

            fully_mapped = 0; 
            if (iblock < lblock) {
                WARN_ON(bh->b_size != blocksize);
                err = get_block(inode, iblock, bh, 0);
                if (err)
                    SetPageError(page);
            }
            if (!buffer_mapped(bh)) {
                zero_user(page, i * blocksize, blocksize);
                if (!err)
                    set_buffer_uptodate(bh);
                continue;
            }
            /*
             * get_block() might have updated the buffer
             * synchronously
             */
            if (buffer_uptodate(bh))
                continue;
        }
        arr[nr++] = bh;
    } while (i++, iblock++, (bh = bh->b_this_page) != head);

    if (fully_mapped)
        SetPageMappedToDisk(page);

    if (!nr) {
    	/*
         * All buffers are uptodate - we can set the page uptodate
         * as well. But not if get_block() returned an error.
         */
        if (!PageError(page))
            SetPageUptodate(page);
        unlock_page(page);
        return 0;
    }

    /* Stage two: lock the buffers */
    for (i = 0; i < nr; i++) {
        bh = arr[i];
        lock_buffer(bh);
        mark_buffer_async_read(bh);
    }

    /*
     * Stage 3: start the IO.  Check for uptodateness
     * inside the buffer lock in case another process reading
     * the underlying blockdev brought it uptodate (the sct fix).
     */
     for (i = 0; i < nr; i++) {
        bh = arr[i];
        if (buffer_uptodate(bh))
            end_buffer_async_read(bh, 1);
        else
        	// 发起bio请求到通用块设备层(gendisk)
            submit_bh(READ, bh);
    }
    return 0;
}

int submit_bh(int rw, struct buffer_head *bh)
{
    return submit_bh_wbc(rw, bh, 0, NULL);
}

static int submit_bh_wbc(int rw, struct buffer_head *bh,
             unsigned long bio_flags, struct writeback_control *wbc)
{
    struct bio *bio;

    BUG_ON(!buffer_locked(bh));
    BUG_ON(!buffer_mapped(bh));
    BUG_ON(!bh->b_end_io);
    BUG_ON(buffer_delay(bh));
    BUG_ON(buffer_unwritten(bh));

    /*
     * Only clear out a write error when rewriting
     */
    if (test_set_buffer_req(bh) && (rw & WRITE))
        clear_buffer_write_io_error(bh);

    /*
     * from here on down, it's all bio -- do the initial mapping,
     * submit_bio -> generic_make_request may further map this bio around
     */
    bio = bio_alloc(GFP_NOIO, 1);

    if (wbc) {
        wbc_init_bio(wbc, bio);
        wbc_account_io(wbc, bh->b_page, bh->b_size);
    }

    bio->bi_iter.bi_sector = bh->b_blocknr * (bh->b_size >> 9);
    bio->bi_bdev = bh->b_bdev;

    bio_add_page(bio, bh->b_page, bh->b_size, bh_offset(bh));
    BUG_ON(bio->bi_iter.bi_size != bh->b_size);

    bio->bi_end_io = end_bio_bh_io_sync;
    bio->bi_private = bh;
    bio->bi_flags |= bio_flags;

    /* Take care of bh's that straddle the end of the device */
    guard_bio_eod(rw, bio);

    if (buffer_meta(bh))
        rw |= REQ_META;
    if (buffer_prio(bh))
        rw |= REQ_PRIO;

    submit_bio(rw, bio);
    return 0;
}
```

6、提交bio请求至通用块设备层（gendisk）

```
/**
 * submit_bio - submit a bio to the block device layer for I/O
 * @rw: whether to %READ or %WRITE, or maybe to %READA (read ahead)
 * @bio: The &struct bio which describes the I/O
 *
 * submit_bio() is very similar in purpose to generic_make_request(), and
 * uses that function to do most of the work. Both are fairly rough
 * interfaces; @bio must be presetup and ready for I/O.
 * 定义：block/blk-core.c
 */
blk_qc_t submit_bio(int rw, struct bio *bio)
{
    bio->bi_rw |= rw;

    /*
     * If it's a regular read/write or a barrier with data attached,
     * go through the normal accounting stuff before submission.
     */
    if (bio_has_data(bio)) {
        unsigned int count;

        if (unlikely(rw & REQ_WRITE_SAME))
            count = bdev_logical_block_size(bio->bi_bdev) >> 9;
        else
            count = bio_sectors(bio);

        if (rw & WRITE) {
            count_vm_events(PGPGOUT, count);
        } else {
            task_io_account_read(bio->bi_iter.bi_size);
            count_vm_events(PGPGIN, count);
        }

        if (unlikely(block_dump)) {
        	char b[BDEVNAME_SIZE];
            printk(KERN_DEBUG "%s(%d): %s block %Lu on %s (%u sectors)\n",
            current->comm, task_pid_nr(current),
                (rw & WRITE) ? "WRITE" : "READ",
                (unsigned long long)bio->bi_iter.bi_sector,
                bdevname(bio->bi_bdev, b),
                count);
        }
    }
	// 这里会将bio添加进gendisk的request_queue
    return generic_make_request(bio);
}
```

7、IO调度器会对gendisk请求队列(request_queue)中的请求进行调度，唤醒块设备绑定的内核线程来fecth  request

```
参考：
	https://zhuanlan.zhihu.com/p/39199521
	https://blog.csdn.net/jasonLee_lijiaqi/article/details/82850689
```



8、剩下的就是具体块设备驱动具体怎么处理请求了

```
EMMC块设备：mmc子系统（drivers/mmc）中card驱动会对fetch的request转换成host controller的命令发送给EMMC的mmc控制器（发送命令之前会设置DMA读取）
```

