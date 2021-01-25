参考：

https://zhuanlan.zhihu.com/p/44267768

https://www.cnblogs.com/alantu2018/p/8461272.html

https://wiki.archlinux.org/index.php/ext4

https://ext4.wiki.kernel.org/index.php?title=Ext4_Howto&oldid=9017（wiki）

https://ext4.wiki.kernel.org/index.php/Ext4_patchsets（patches）

### 1、source

`fs/ext4`

### 2、data struct

- 1、ext4需要注册到VFS的file_system_type

```c
// ext4的定义如下：
static struct file_system_type ext4_fs_type = {
        .owner          = THIS_MODULE,
        .name           = "ext4",
        .mount          = ext4_mount,			// ext4格式分区挂载的函数
        .kill_sb        = kill_block_super,		// ext4格式分区卸载函数
        .fs_flags       = FS_REQUIRES_DEV,		// ext4文件系统必须在物理设备	
};
MODULE_ALIAS_FS("ext4");
```

### 3、function

- 1、ext4  init

```c
// file ：fs/ext4/super.c
module_init(ext4_init_fs);
```

- 2、ext4 registe to vfs

```c
// file ：fs/ext4/super.c
register_filesystem(&ext4_fs_type);
```

