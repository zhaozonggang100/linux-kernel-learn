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
434         return new_sync_read(file, buf, count, pos);
435     else
436         return -EINVAL;
437 }
```
