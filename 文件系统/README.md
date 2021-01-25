### 1、重要结构

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
        // fsopen系统调用内部创建FS_CONTEXT_FOR_MOUNT的fs上下文
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
    	// 此文件系统类型的文件系统超级块结构都串连在这个表头下
        struct hlist_head fs_supers;

        struct lock_class_key s_lock_key;
        struct lock_class_key s_umount_key;
        struct lock_class_key s_vfs_rename_key;
        struct lock_class_key s_writers_key[SB_FREEZE_LEVELS];

        struct lock_class_key i_lock_key;
        struct lock_class_key i_mutex_key;
        struct lock_class_key i_mutex_dir_key;
};

// 所有已经注册的文件系统都挂载到该指针指向的链表中
static struct file_system_type *file_systems;
```

- 2、文件系统上下文，比较新的内核支持,XFS中有定义，ext4没有定义
```c
/*
 * Filesystem context for holding the parameters used in the creation or
 * reconfiguration of a superblock.
 *
 * Superblock creation fills in ->root whereas reconfiguration begins with this
 * already set.
 *
 * See Documentation/filesystems/mount_api.rst
 */
struct fs_context {
        const struct fs_context_operations *ops;
        struct mutex            uapi_mutex;     /* Userspace access mutex */
        struct file_system_type *fs_type;
        void                    *fs_private;    /* The filesystem's context */
        void                    *sget_key;
        struct dentry           *root;          /* The root and superblock */
        struct user_namespace   *user_ns;       /* The user namespace for this mount */
        struct net              *net_ns;        /* The network namespace for this mount */
        const struct cred       *cred;          /* The mounter's credentials */
        struct p_log            log;            /* Logging buffer */
        const char              *source;        /* The source name (eg. dev path) */
        void                    *security;      /* Linux S&M options */
        void                    *s_fs_info;     /* Proposed s_fs_info */
        unsigned int            sb_flags;       /* Proposed superblock flags (SB_*) */
        unsigned int            sb_flags_mask;  /* Superblock flags that were changed */
        unsigned int            s_iflags;       /* OR'd with sb->s_iflags */
        unsigned int            lsm_flags;      /* Information flags from the fs to the LSM */
        enum fs_context_purpose purpose:8;
        enum fs_context_phase   phase:8;        /* The phase the context is in */
        bool                    need_free:1;    /* Need to call ops->free() */
        bool                    global:1;       /* Goes into &init_user_ns */
        bool                    oldapi:1;       /* Coming from mount(2) */
};

struct fs_context_operations {
        void (*free)(struct fs_context *fc);
        int (*dup)(struct fs_context *fc, struct fs_context *src_fc);
        int (*parse_param)(struct fs_context *fc, struct fs_parameter *param);
        int (*parse_monolithic)(struct fs_context *fc, void *data);
        int (*get_tree)(struct fs_context *fc);
        int (*reconfigure)(struct fs_context *fc);
};
```

### 2、重要函数

- 1、注册一个具体的文件系统到VFS

```c
/**
 *		file : fs/filesystems.c
 *      register_filesystem - register a new filesystem
 *      @fs: the file system structure
 *      
 *      Adds the file system passed to the list of file systems the kernel
 *      is aware of for mount and other syscalls. Returns 0 on success,
 *      or a negative errno code on an error.
 *      
 *      The &struct file_system_type that is passed is linked into the kernel 
 *      structures and must not be freed until the file system has been
 *      unregistered.
 */     
        
int register_filesystem(struct file_system_type * fs)
{       
        int res = 0; 
        struct file_system_type ** p;
        
        if (fs->parameters &&
            !fs_validate_description(fs->name, fs->parameters))
                return -EINVAL;

        BUG_ON(strchr(fs->name, '.'));
        if (fs->next)
                return -EBUSY;
        write_lock(&file_systems_lock);
        p = find_filesystem(fs->name, strlen(fs->name));
        if (*p)
                res = -EBUSY;	// 具体的某个文件系统已经在全局的file_system指针指向的链表中
        else
                *p = fs;		// find_filesystem中将具体的某个文件系统的file_system_type挂载到全局的file_system指向的链表的最后一个节点的next成员中
        write_unlock(&file_systems_lock);
        return res;
}
```

- 2、mount时通过文件系统名字获取对应的file_system_type结构

```c
struct file_system_type *get_fs_type(const char *name)
{       
        struct file_system_type *fs;
        const char *dot = strchr(name, '.');
        int len = dot ? dot - name : strlen(name);
  
        fs = __get_fs_type(name, len);
        if (!fs && (request_module("fs-%.*s", len, name) == 0)) {
                fs = __get_fs_type(name, len);
                if (!fs)
                        pr_warn_once("request_module fs-%.*s succeeded, but still no fs?\n",
                                     len, name);
        }

        if (dot && fs && !(fs->fs_flags & FS_HAS_SUBTYPE)) {
                put_filesystem(fs);
                fs = NULL;
        }       
        return fs;
}
```

### 3、mount系统调用  

- ref  
        1、https://zhuanlan.zhihu.com/p/93592262（Red Hat 工程师-新以代mount系统调用实现）