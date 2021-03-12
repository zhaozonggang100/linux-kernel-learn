### 1、概要
&emsp;&emsp;Linux内核支持多种文件系统类型（file_system_type），要使用某种类型的文件系统，必须调用**register_filesystem**将该类型的文件系统注册给VFS，不用时调用**unregister_filesystem**从VFS中注销。  

&emsp;&emsp;所有在系统中注册过的文件系统类型对象链入到一个**单链表**中，**file_systems**(fs/filesystem.c)变量指向链表的第一项，每个文件系统类型的next成员指向链表的下一项。

&emsp;&emsp;文件系统类型注册的目的是向VFS提供**mount**、**kill_sb**回调函数，分别在Linux装载和卸载这种类型的文件系统实例时调用。

### 2、**data struct**

```c++
struct file_system_type {
        // 文件系统类型名字，eg：ext4
        const char *name;
        // 文件系统类型标识
        int fs_flags;
#define FS_REQUIRES_DEV         1 
#define FS_BINARY_MOUNTDATA     2
#define FS_HAS_SUBTYPE          4
#define FS_USERNS_MOUNT         8       /* Can be mounted by userns root */
#define FS_USERNS_DEV_MOUNT     16 /* A userns mount does not imply MNT_NODEV */
#define FS_USERNS_VISIBLE       32      /* FS must already be visible */
#define FS_RENAME_DOES_D_MOVE   32768   /* FS will handle d_move() during rename() internally. */
        // 在这种类型的文件系统实例被装载时调用
        struct dentry *(*mount) (struct file_system_type *, int,
                       const char *, void *); 
        struct dentry *(*mount2) (struct vfsmount *, struct file_system_type *, int,
                               const char *, void *);
        void *(*alloc_mnt_data) (void);
        // 在这种类型的文件系统实例被卸载时调用
        void (*kill_sb) (struct super_block *);
        struct module *owner;
        // 指向文件系统类型的下一个元素，eg：ext4的该成员指向xfs的file_system_type结构
        struct file_system_type * next;
        // 同一种类型的文件系统的超级块实例的链表的表头，超级块（super_block）通过成员<struct hlist_node       s_instances;>链接进来
        struct hlist_head fs_supers;

        // 下面的所有成员用于调试锁依赖性
        struct lock_class_key s_lock_key;
        struct lock_class_key s_umount_key;
        struct lock_class_key s_vfs_rename_key;
        struct lock_class_key s_writers_key[SB_FREEZE_LEVELS];
        
        struct lock_class_key i_lock_key;
        struct lock_class_key i_mutex_key;
        struct lock_class_key i_mutex_dir_key;
};
```