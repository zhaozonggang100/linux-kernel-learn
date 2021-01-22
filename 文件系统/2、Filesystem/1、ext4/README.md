参考：

https://zhuanlan.zhihu.com/p/44267768

https://www.cnblogs.com/alantu2018/p/8461272.html

https://wiki.archlinux.org/index.php/ext4

https://ext4.wiki.kernel.org/index.php?title=Ext4_Howto&oldid=9017（wiki）

https://ext4.wiki.kernel.org/index.php/Ext4_patchsets（patches）

### 1、source

`fs/ext4`



### 2、data struct

- 1、每种类型文件系统都要注册给VFS的结构

```c
struct file_system_type {
        const char *name;				// 文件系统名字，ext4文件系统就是ext4
    	/*
    		说明文件系统的类型，下面的宏定义代表了它的几种类型：
    			FS_REQUIRES_DEV: 文件系统必须在物理设备上。
				FS_BINARY_MOUNTDATA: mount此文件系统时（参见mount_fs函数 - fs/super.c）需要使用二进制数据结构的mount data（如每个位域都有固定
				的位置和意义），常见的nfs使用这种mount data（参见struct nfs_mount_data结构 - include/uapi/linux/nfs_mount.h）。
				FS_HAS_SUBTYPE: 文件系统含有子类型，最常见的就是FUSE，FUSE本是不是真正的文件系统，所以要通过子文件系统类型来区别通过FUSE接口实现的不
				同文件系统。
				FS_USERNS_MOUNT: 文件系统每次挂载都后都是不同的user namespace，如用于devpts。
				FS_RENAME_DOES_D_MOVE: 文件系统将把重命名操作reame()直接按照移动操作d_move()来处理，主要用于网络文件系统。
    	*/
        int fs_flags;
#define FS_REQUIRES_DEV         1 
#define FS_BINARY_MOUNTDATA     2
#define FS_HAS_SUBTYPE          4
#define FS_USERNS_MOUNT         8       /* Can be mounted by userns root */
#define FS_DISALLOW_NOTIFY_PERM 16      /* Disable fanotify permission events */
#define FS_THP_SUPPORT          8192    /* Remove once all fs converted */
#define FS_RENAME_DOES_D_MOVE   32768   /* FS will handle d_move() during rename() internally. */
        int (*init_fs_context)(struct fs_context *);
        const struct fs_parameter_spec *parameters;
    	// 通过register_filesystem将file_system_type注册给VFS，挂载文件系统时根据指定的type，调用到ext4的mount函数
    	// mount的过程会从指定的块设备读取super_block，这个流程需要单独分析
        struct dentry *(*mount) (struct file_system_type *, int, const char *, void *);
     	// 卸载指定挂载点的文件系统时调用，会删除内存中对应的super_block
        void (*kill_sb) (struct super_block *);
    	// 指向实现这个文件系统的模块，通常为THIS_MODULE宏。
        struct module *owner;
    	// 指向文件系统类型链表的下一个文件系统类型
        struct file_system_type * next;
        struct hlist_head fs_supers;

        struct lock_class_key s_lock_key;
        struct lock_class_key s_umount_key;
        struct lock_class_key s_vfs_rename_key;
        struct lock_class_key s_writers_key[SB_FREEZE_LEVELS];

        struct lock_class_key i_lock_key;
        struct lock_class_key i_mutex_key;
        struct lock_class_key i_mutex_dir_key;
};

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
// file : fs/filesystems.c
int register_filesystem(struct file_system_type * fs);
```

