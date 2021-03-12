### 1、info

- 1、vfsmount反应了一个`已经挂载`的文件系统实例  

- 2、一个文件的位置由`<vfsmount,dentry>`决定

- 3、一个分区被装载一次就会生成一个**vfsmount**结构

### 2、data struct

```c++
struct vfsmount {
        // 当前文件系统根目录
        struct dentry *mnt_root;        /* root of the mounted tree */
        // 文件系统对应的VFS超级块对象
        struct super_block *mnt_sb;     /* pointer to superblock */
        // 挂载标志
        int mnt_flags;
} __randomize_layout;
```