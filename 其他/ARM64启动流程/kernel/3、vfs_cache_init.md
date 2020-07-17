### 1、Linux文件系统的初始化

初期会执行vfs_cache_init初始化vfs，后期会执行initrd挂载真正的磁盘文件系统

### 2、vfs_cache_init执行流程

```c
1、dcache_init：dentry的slub缓存和哈希表初始化
2、inode_init：inode结构的slub缓存和哈希表初始化
3、files_init：file结构的slub缓存初始化
4、files_maxfiles_init
5、mnt_init
	https://www.cnblogs.com/awsqsh/articles/4358285.html
	https://www.cnblogs.com/arnoldlu/p/10986583.html
	1、初始化mnt_cache的slub缓存，vfs_mount
	2、初始化mount_hashtable挂载哈希表
	3、初始化mountpoint_hashtable挂载点哈希表
	4、kernfs_init
	5、sysfs_init：sysfs初始化
	6、init_rootfs：注册虚拟根文件系统，registe_filesystem
        int __init init_rootfs(void)
        {   
            int err = register_filesystem(&rootfs_fs_type);

            if (err)
                return err;

            if (IS_ENABLED(CONFIG_TMPFS) && !saved_root_name[0] &&
                (!root_fs_names || strstr(root_fs_names, "tmpfs"))) {
                /*
                	没有指定saved_root_name并且root_fs_names为tmpfs时候，初始化tmpfs文件系统
                	初始化tmpfs文件系统。
                */
                err = shmem_init();
                // 后面rootfs_mount()会需要判断是使用tmpfs还是ramfs作为文件系统类型
                is_tmpfs = true;
            } else {
                err = init_ramfs_fs();	// 初始化ramfs
            }

            if (err)
                unregister_filesystem(&rootfs_fs_type);

            return err;
        }
	7、init_mount_tree：挂载虚拟根文件系统
		static void __init init_mount_tree(void)
        {
            struct vfsmount *mnt;
            struct mnt_namespace *ns;
            struct path root;
            struct file_system_type *type;

        	// 获取rootfs对应的file_system_type，这里对应的是ramfs操作函数。
            type = get_fs_type("rootfs");
            if (!type)
                panic("Can't find rootfs type");
        	// 这里会调用mount_fs()，进而调用rootfs_fs_type->mount()，即rootfs_mount()
            mnt = vfs_kern_mount(type, 0, "rootfs", NULL);
            put_filesystem(type);
            if (IS_ERR(mnt))
                panic("Can't create rootfs");

            ns = create_mnt_ns(mnt);
            if (IS_ERR(ns)) 
                panic("Can't allocate initial namespace");

            init_task.nsproxy->mnt_ns = ns;
            get_mnt_ns(ns);

            root.mnt = mnt;
            root.dentry = mnt->mnt_root;
            mnt->mnt_flags |= MNT_LOCKED;

            set_fs_pwd(current->fs, &root);
        	// 将当前的文件系统盘配置为根文件系统，也就是ramfs设置为根文件系统
            set_fs_root(current->fs, &root);
        } 
		
		static struct dentry *rootfs_mount(struct file_system_type *fs_type,
            int flags, const char *dev_name, void *data)
        {   
            static unsigned long once;
            void *fill = ramfs_fill_super;

            if (test_and_set_bit(0, &once))
                return ERR_PTR(-ENODEV);

            if (IS_ENABLED(CONFIG_TMPFS) && is_tmpfs)
                fill = shmem_fill_super;

            return mount_nodev(fs_type, flags, data, fill);
        }
6、bdev_cache_init：初始化并挂载bdevfs
7、chrdev_init：字符设备相关
```

### 3、疑问：为什么sysfs比rootfs挂载还早？？？

```
1、sysfs比rootfs还早挂载，这是Linux为了全面展示设备模型，根文件系统有两种，一种是虚拟根文件系统，另外一种是真实的根文件系统。一般情况下，会首先在虚拟的根文件系统中做一部分工作，然后切换到真实的根文件系统下面。

笼统的来说，虚拟的根文件系统包括三种类型，即Initramfs、cpio-initrd和image-initrd。

此时的sysfs还没有挂载到/sys目录，rootfs和sysfs都只是在内存的/节点中，当磁盘挂载之后，会切换rootfs的挂载点到磁盘的/节点，同时将sysfs挂载到/sys节点上
    		
2、在strat_kernel初始化kernel的时候最后会调用rest_init，在rest_init中会调用驱动通过module_init或其他xxx_init注册到链接脚本制定的init段中函数，init段会根据优先级执行不同的init段中的函数，详情可以google Linux的module_init机制。
	start_kernel()--->rest_init()-->kernel_init->kernel_init_freeable()-->do_basic_setup()->do_initcalls()

3、在init/initramfs.c中通过rootfs_initcall(populate_rootfs)注册了populate_rootfs
	populate_rootfs函数加载initrd，initrd文件系统提供了init程序，在linux初始化阶段的后期会跳转到init程序，由该程序负责加载驱动程序和挂载磁盘文件系统以及其他的初始化工作。
	CONFIG_BLK_DEV_RAM代表直接用ram代替硬盘，比如ramdisk
```

### 4、Linux中的initrd

https://blog.csdn.net/kevin_hcy/article/details/17663341

- initrd分类

```
1、无Initrd
2、Linux Kernel和Initrd打包
3、Linux Kernel和Initrd分离
4、RAMDisk Initrd			// 最常用
```

### 5、populate_rootfs函数如何挂载

```c
/*
	https://blog.csdn.net/skyflying2012/article/details/8764486
	http://blog.chinaunix.net/uid-26694208-id-3078013.html
	下面的初始化函数会将函数通过gcc编译器将函数放到制定的段中
	arm64链接脚本：./arch/arm64/kernel/vmlinux.lds
	上面的路径为编译输出路径，链接脚本是通过vmlinux.lds.S编译生成的
	里面包含了initcall0.init、initcall1.init、...、initcallrootfs.init、initcall7.init
*/
#define __define_initcall(fn, id) \
    static initcall_t __initcall_name(fn, id) __used \
    __attribute__((__section__(".initcall" #id ".init"))) = fn;		// 定义一个函数指针__initcall_name把fn赋值给它，然后把这个函数指针放在.initcallid.init section(#号是连接符)

#define rootfs_initcall(fn)     __define_initcall(fn, rootfs)

/*
	__init修饰的函数会放到链接脚本的.init.text段
*/
static int __init populate_rootfs(void)
{
    char *err;

    if (do_skip_initramfs) {
        if (initrd_start)
            free_initrd();
        return default_rootfs();
    }   

    /* Load the built in initramfs */
    err = unpack_to_rootfs(__initramfs_start, __initramfs_size);
    if (err)
        panic("%s", err); /* Failed to decompress INTERNAL initramfs */
    /* If available load the bootloader supplied initrd */
    if (initrd_start && !IS_ENABLED(CONFIG_INITRAMFS_FORCE)) {
#ifdef CONFIG_BLK_DEV_RAM
        int fd; 
        printk(KERN_INFO "Trying to unpack rootfs image as initramfs...\n");
        err = unpack_to_rootfs((char *)initrd_start,
            initrd_end - initrd_start);
        if (!err) {
            free_initrd();
            goto done;
        } else {
            clean_rootfs();
            unpack_to_rootfs(__initramfs_start, __initramfs_size);
        }
        printk(KERN_INFO "rootfs image is not initramfs (%s)"
                "; looks like an initrd\n", err);
        fd = sys_open("/initrd.image",
                  O_WRONLY|O_CREAT, 0700);
        if (fd >= 0) {
            ssize_t written = xwrite(fd, (char *)initrd_start,
                        initrd_end - initrd_start);

            if (written != initrd_end - initrd_start)
                pr_err("/initrd.image: incomplete write (%zd != %ld)\n",
                       written, initrd_end - initrd_start);

            sys_close(fd);
            free_initrd();
        }
    done:
        /* empty statement */;
        #else
        printk(KERN_INFO "Unpacking initramfs...\n");
        err = unpack_to_rootfs((char *)initrd_start,
            initrd_end - initrd_start);
        if (err)
            printk(KERN_EMERG "Initramfs unpacking failed: %s\n", err);
        free_initrd();
#endif
    }
    flush_delayed_fput();
    /*
     * Try loading default modules from initramfs.  This gives
     * us a chance to load before device_initcalls.
     */
    load_default_modules();

    return 0;
}
/*
	这句话的意思就是把populate_rootfs函数放在initcallrootfs.init段
	module_init的定义：
		#define module_init(x)  __initcall(x);
		#define __initcall(fn) device_initcall(fn)
		
		#define pure_initcall(fn)       __define_initcall(fn, 0)
        #define core_initcall(fn)       __define_initcall(fn, 1)
        #define core_initcall_sync(fn)      __define_initcall(fn, 1s)
        #define postcore_initcall(fn)       __define_initcall(fn, 2)
        #define postcore_initcall_sync(fn)  __define_initcall(fn, 2s)
        #define arch_initcall(fn)       __define_initcall(fn, 3)
        #define arch_initcall_sync(fn)      __define_initcall(fn, 3s)
        #define subsys_initcall(fn)     __define_initcall(fn, 4)
        #define subsys_initcall_sync(fn)    __define_initcall(fn, 4s)
        #define fs_initcall(fn)         __define_initcall(fn, 5)
        #define fs_initcall_sync(fn)        __define_initcall(fn, 5s)
        #define rootfs_initcall(fn)     __define_initcall(fn, rootfs)
        #define device_initcall(fn)     __define_initcall(fn, 6)
        #define device_initcall_sync(fn)    __define_initcall(fn, 6s)
        #define late_initcall(fn)       __define_initcall(fn, 7)
        #define late_initcall_sync(fn)      __define_initcall(fn, 7s)
     所以module_init会将函数放到initcall6.init段中
*/
rootfs_initcall(populate_rootfs);
```

- ramdisk原理

```c
// https://www.cnblogs.com/arnoldlu/p/10986583.html
```

