### 1、概要

- 1、作用  

&emsp;&emsp;inode表示文件系统对象的元数据，dentry表示文件系统对象在文件系统树中的位置，dentry和inode是多对1的关系，每个dentry只有一个inode，由`d_inode`域指向；而一个inode可能对应多个dentry（比如硬链接），将这些dentry组成以i_dentry的链表。  

&emsp;&emsp;所有的dentry链入全局的dentry_hashtable中（root dentry除外），这个哈希表为了方便查找给定目录下的文件。

### 2、data struct

```c++
// file：include/linux/dcache.h
struct dentry {
        /* RCU lookup touched fields */
        unsigned int d_flags;           /* protected by d_lock */
        seqcount_spinlock_t d_seq;      /* per dentry seqlock */
        // 链入到全局dentry_hashtable或者超级块的匿名dentry哈希表的"连接件"，根dentry除外
        struct hlist_bl_node d_hash;    /* lookup hash list */
        // 父目录的dentry，如果是根目录，则指向自己
        struct dentry *d_parent;        /* parent directory */
        // dentry表示的文件名、长度、哈希值
        struct qstr d_name;
        // dentry关联的inode描述符
        struct inode *d_inode;          /* Where the name belongs to - NULL is
                                         * negative */
        // 保存短文件名，如果文件名长度超过36，则另外分配空间
        unsigned char d_iname[DNAME_INLINE_LEN];        /* small names */

        /* Ref lookup also touches following */
        struct lockref d_lockref;       /* per-dentry lock and refcount */
        const struct dentry_operations *d_op;
        // dentry所属的文件系统超级块对象
        struct super_block *d_sb;       /* The root of the dentry tree */
        // 被d_revalidate函数用来判断dentry是否还有有效的数据
        unsigned long d_time;           /* used by d_revalidate */
        // 指向具体文件系统的dentry
        void *d_fsdata;                 /* fs-specific data */

        union {
                // 文件系统未使用的dentry链表连接件，super_block的s_dentry_lru域为链表头
                struct list_head d_lru;         /* LRU list */
                wait_queue_head_t *d_wait;      /* in-lookup ones only */
        };
        // 子dentry通过该域链入父dentry的子dentry链表中
        struct list_head d_child;       /* child of parent list */
        // 子dentry链表的链表头
        struct list_head d_subdirs;     /* our children */
        /*
         * d_alias and d_rcu can share memory
         */
        union {
                // 链入到所属inode的i_dentry（别名）链表的"连接件"
                struct hlist_node d_alias;      /* inode alias list */
                struct hlist_bl_node d_in_lookup_hash;  /* only for in-lookup ones */
                struct rcu_head d_rcu;
        } d_u;
} __randomize_layout;

/*
 * "quick string" -- eases parameter passing, but more importantly
 * saves "metadata" about the string (ie length and the hash).
 *
 * hash comes first so it snuggles against d_parent in the
 * dentry.
 */
struct qstr {
        union {
                struct {
                        HASH_LEN_DECLARE;
                };
                u64 hash_len;
        };
        const unsigned char *name;
};

struct dentry_operations {
        // 路径查找时找到目录项之后确认目录项对应的磁盘目录项是否存在
        int (*d_revalidate)(struct dentry *, unsigned int);
        int (*d_weak_revalidate)(struct dentry *, unsigned int);
        int (*d_hash)(const struct dentry *, struct qstr *);
        int (*d_compare)(const struct dentry *,
                        unsigned int, const char *, const struct qstr *);
        int (*d_delete)(const struct dentry *);
        int (*d_init)(struct dentry *);
        void (*d_release)(struct dentry *);
        void (*d_prune)(struct dentry *);
        void (*d_iput)(struct dentry *, struct inode *);
        char *(*d_dname)(struct dentry *, char *, int);
        struct vfsmount *(*d_automount)(struct path *);
        int (*d_manage)(const struct path *, bool);
        struct dentry *(*d_real)(struct dentry *, const struct inode *);
} ____cacheline_aligned;
```