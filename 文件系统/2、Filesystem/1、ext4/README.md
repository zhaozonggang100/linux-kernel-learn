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

```c++
static struct file_system_type ext4_fs_type = {
        .owner          = THIS_MODULE,
        .name           = "ext4",
        .mount          = ext4_mount,			// ext4格式分区挂载的函数
        .kill_sb        = kill_block_super,		// ext4格式分区卸载函数
        .fs_flags       = FS_REQUIRES_DEV,		// ext4文件系统必须在物理设备	
};
MODULE_ALIAS_FS("ext4");
```

- 2、ext4的磁盘中的superblock**定义**

```c++
// file ： fs/ext4/ext4.h
struct ext4_super_block {
/*00*/  __le32  s_inodes_count;         /* Inodes count */
        __le32  s_blocks_count_lo;      /* Blocks count */
        __le32  s_r_blocks_count_lo;    /* Reserved blocks count */
        __le32  s_free_blocks_count_lo; /* Free blocks count */
/*10*/  __le32  s_free_inodes_count;    /* Free inodes count */
        __le32  s_first_data_block;     /* First Data Block */
        __le32  s_log_block_size;       /* Block size */
        __le32  s_log_cluster_size;     /* Allocation cluster size */
/*20*/  __le32  s_blocks_per_group;     /* # Blocks per group */
        __le32  s_clusters_per_group;   /* # Clusters per group */
        __le32  s_inodes_per_group;     /* # Inodes per group */
        __le32  s_mtime;                /* Mount time */
/*30*/  __le32  s_wtime;                /* Write time */
        __le16  s_mnt_count;            /* Mount count */
        __le16  s_max_mnt_count;        /* Maximal mount count */
        __le16  s_magic;                /* Magic signature */
        __le16  s_state;                /* File system state */
        __le16  s_errors;               /* Behaviour when detecting errors */
        __le16  s_minor_rev_level;      /* minor revision level */
/*40*/  __le32  s_lastcheck;            /* time of last check */
        __le32  s_checkinterval;        /* max. time between checks */
        __le32  s_creator_os;           /* OS */
        __le32  s_rev_level;            /* Revision level */
/*50*/  __le16  s_def_resuid;           /* Default uid for reserved blocks */
        __le16  s_def_resgid;           /* Default gid for reserved blocks */
        /*
         * These fields are for EXT4_DYNAMIC_REV superblocks only.
         *
         * Note: the difference between the compatible feature set and
         * the incompatible feature set is that if there is a bit set
         * in the incompatible feature set that the kernel doesn't
         * know about, it should refuse to mount the filesystem.
         *
         * e2fsck's requirements are more strict; if it doesn't know
         * about a feature in either the compatible or incompatible
         * feature set, it must abort and not try to meddle with
         * things it doesn't understand...
         */
        __le32  s_first_ino;            /* First non-reserved inode */
        __le16  s_inode_size;           /* size of inode structure */
        __le16  s_block_group_nr;       /* block group # of this superblock */
        __le32  s_feature_compat;       /* compatible feature set */
/*60*/  __le32  s_feature_incompat;     /* incompatible feature set */
        __le32  s_feature_ro_compat;    /* readonly-compatible feature set */
/*68*/  __u8    s_uuid[16];             /* 128-bit uuid for volume */
/*78*/  char    s_volume_name[16];      /* volume name */
/*88*/  char    s_last_mounted[64];     /* directory where last mounted */
/*C8*/  __le32  s_algorithm_usage_bitmap; /* For compression */
        /*
         * Performance hints.  Directory preallocation should only
         * happen if the EXT4_FEATURE_COMPAT_DIR_PREALLOC flag is on.
         */
        __u8    s_prealloc_blocks;      /* Nr of blocks to try to preallocate*/
        __u8    s_prealloc_dir_blocks;  /* Nr to preallocate for dirs */
        __le16  s_reserved_gdt_blocks;  /* Per group desc for online growth */
        /*
         * Journaling support valid if EXT4_FEATURE_COMPAT_HAS_JOURNAL set.
         */
/*D0*/  __u8    s_journal_uuid[16];     /* uuid of journal superblock */
/*E0*/  __le32  s_journal_inum;         /* inode number of journal file */
        __le32  s_journal_dev;          /* device number of journal file */
        __le32  s_last_orphan;          /* start of list of inodes to delete */
        __le32  s_hash_seed[4];         /* HTREE hash seed */
        __u8    s_def_hash_version;     /* Default hash version to use */
        __u8    s_jnl_backup_type;
        __le16  s_desc_size;            /* size of group descriptor */
/*100*/ __le32  s_default_mount_opts;
        __le32  s_first_meta_bg;        /* First metablock block group */
        __le32  s_mkfs_time;            /* When the filesystem was created */
        __le32  s_jnl_blocks[17];       /* Backup of the journal inode */
        /* 64bit support valid if EXT4_FEATURE_COMPAT_64BIT */
/*150*/ __le32  s_blocks_count_hi;      /* Blocks count */
        __le32  s_r_blocks_count_hi;    /* Reserved blocks count */
        __le32  s_free_blocks_count_hi; /* Free blocks count */
        __le16  s_min_extra_isize;      /* All inodes have at least # bytes */
        __le16  s_want_extra_isize;     /* New inodes should reserve # bytes */
        __le32  s_flags;                /* Miscellaneous flags */
        __le16  s_raid_stride;          /* RAID stride */
        __le16  s_mmp_update_interval;  /* # seconds to wait in MMP checking */
        __le64  s_mmp_block;            /* Block for multi-mount protection */
        __le32  s_raid_stripe_width;    /* blocks on all data disks (N*stride)*/
        __u8    s_log_groups_per_flex;  /* FLEX_BG group size */
        __u8    s_checksum_type;        /* metadata checksum algorithm used */
        __u8    s_encryption_level;     /* versioning level for encryption */
        __u8    s_reserved_pad;         /* Padding to next 32bits */
        __le64  s_kbytes_written;       /* nr of lifetime kilobytes written */
        __le32  s_snapshot_inum;        /* Inode number of active snapshot */
        __le32  s_snapshot_id;          /* sequential ID of active snapshot */
        __le64  s_snapshot_r_blocks_count; /* reserved blocks for active
                                              snapshot's future use */
        __le32  s_snapshot_list;        /* inode number of the head of the
                                           on-disk snapshot list */
#define EXT4_S_ERR_START offsetof(struct ext4_super_block, s_error_count)
        __le32  s_error_count;          /* number of fs errors */
        __le32  s_first_error_time;     /* first time an error happened */
        __le32  s_first_error_ino;      /* inode involved in first error */
        __le64  s_first_error_block;    /* block involved of first error */
        __u8    s_first_error_func[32]; /* function where the error happened */
        __le32  s_first_error_line;     /* line number where error happened */
        __le32  s_last_error_time;      /* most recent time of an error */
        __le32  s_last_error_ino;       /* inode involved in last error */
        __le32  s_last_error_line;      /* line number where error happened */
        __le64  s_last_error_block;     /* block involved of last error */
        __u8    s_last_error_func[32];  /* function where the error happened */
#define EXT4_S_ERR_END offsetof(struct ext4_super_block, s_mount_opts)
        __u8    s_mount_opts[64];
        __le32  s_usr_quota_inum;       /* inode for tracking user quota */
        __le32  s_grp_quota_inum;       /* inode for tracking group quota */
        __le32  s_overhead_clusters;    /* overhead blocks/clusters in fs */
        __le32  s_backup_bgs[2];        /* groups with sparse_super2 SBs */
        __u8    s_encrypt_algos[4];     /* Encryption algorithms in use  */
        __u8    s_encrypt_pw_salt[16];  /* Salt used for string2key algorithm */
        __le32  s_lpf_ino;              /* Location of the lost+found inode */
        __le32  s_prj_quota_inum;       /* inode for tracking project quota */
        __le32  s_checksum_seed;        /* crc32c(uuid) if csum_seed set */
        __le32  s_reserved[98];         /* Padding to the end of the block */
        __le32  s_checksum;             /* crc32c(superblock) */
};
```

- 3、ext4的superblock operations结构初始化  

```c++
// file ： fs/ext4/super.c
static const struct super_operations ext4_sops = {
        .alloc_inode    = ext4_alloc_inode,
        .destroy_inode  = ext4_destroy_inode,
        .write_inode    = ext4_write_inode,
        .dirty_inode    = ext4_dirty_inode,
        .drop_inode     = ext4_drop_inode,
        .evict_inode    = ext4_evict_inode,
        // VFS释放ext4的超级块时调用
        .put_super      = ext4_put_super,
        // VFS写出和EXT4超级块关联的脏数据时调用
        .sync_fs        = ext4_sync_fs,
        // 锁住ext4，强制使它进一致性状态时调用，LVM使用这种方式
        .freeze_fs      = ext4_freeze,
        .unfreeze_fs    = ext4_unfreeze,
        .statfs         = ext4_statfs,
        .remount_fs     = ext4_remount,
        .show_options   = ext4_show_options,
#ifdef CONFIG_QUOTA
        .quota_read     = ext4_quota_read,
        .quota_write    = ext4_quota_write,
        .get_dquots     = ext4_get_dquots,
#endif
        .bdev_try_to_free_page = bdev_try_to_free_page,
};
```

- 4、ext4内存中的超级块  

```c++
// file: fs/ext4/ext4.h
/*
 * fourth extended-fs super-block data in memory
 */
struct ext4_sb_info {
        unsigned long s_desc_size;      /* Size of a group descriptor in bytes */
        unsigned long s_inodes_per_block;/* Number of inodes per block */
        unsigned long s_blocks_per_group;/* Number of blocks in a group */
        unsigned long s_clusters_per_group; /* Number of clusters in a group */
        unsigned long s_inodes_per_group;/* Number of inodes in a group */
        unsigned long s_itb_per_group;  /* Number of inode table blocks per group */
        unsigned long s_gdb_count;      /* Number of group descriptor blocks */
        unsigned long s_desc_per_block; /* Number of group descriptors per block */
        ext4_group_t s_groups_count;    /* Number of groups in the fs */
        ext4_group_t s_blockfile_groups;/* Groups acceptable for non-extent files */
        unsigned long s_overhead;  /* # of fs overhead clusters */
        unsigned int s_cluster_ratio;   /* Number of blocks per cluster */
        unsigned int s_cluster_bits;    /* log2 of s_cluster_ratio */
        loff_t s_bitmap_maxbytes;       /* max bytes for bitmap files */
        struct buffer_head * s_sbh;     /* Buffer containing the super block */
        struct ext4_super_block *s_es;  /* Pointer to the super block in the buffer */
        struct buffer_head **s_group_desc;
        unsigned int s_mount_opt;
        unsigned int s_mount_opt2;
        unsigned int s_mount_flags;
        unsigned int s_def_mount_opt;
        ext4_fsblk_t s_sb_block;
        atomic64_t s_resv_clusters;
        kuid_t s_resuid;
        kgid_t s_resgid;
        unsigned short s_mount_state;
        unsigned short s_pad;
        int s_addr_per_block_bits;
        int s_desc_per_block_bits;
        int s_inode_size;
        int s_first_ino;
        unsigned int s_inode_readahead_blks;
        unsigned int s_inode_goal;
        spinlock_t s_next_gen_lock;
        u32 s_next_generation;
        u32 s_hash_seed[4];
        int s_def_hash_version;
        int s_hash_unsigned;    /* 3 if hash should be signed, 0 if not */
        struct percpu_counter s_freeclusters_counter;
        struct percpu_counter s_freeinodes_counter;
        struct percpu_counter s_dirs_counter;
        struct percpu_counter s_dirtyclusters_counter;
        struct blockgroup_lock *s_blockgroup_lock;
        struct proc_dir_entry *s_proc;
        struct kobject s_kobj;
        struct completion s_kobj_unregister;
        struct super_block *s_sb;

        /* Journaling */
        struct journal_s *s_journal;
        struct list_head s_orphan;
        struct mutex s_orphan_lock;
        unsigned long s_resize_flags;           /* Flags indicating if there
                                                   is a resizer */
        unsigned long s_commit_interval;
        u32 s_max_batch_time;
        u32 s_min_batch_time;
        struct block_device *journal_bdev;
#ifdef CONFIG_QUOTA
        char *s_qf_names[EXT4_MAXQUOTAS];       /* Names of quota files with journalled quota */
        int s_jquota_fmt;                       /* Format of quota to use */
#endif
        unsigned int s_want_extra_isize; /* New inodes should reserve # bytes */
        struct rb_root system_blks;

#ifdef EXTENTS_STATS
        /* ext4 extents stats */
        unsigned long s_ext_min;
        unsigned long s_ext_max;
        unsigned long s_depth_max;
        spinlock_t s_ext_stats_lock;
        unsigned long s_ext_blocks;
        unsigned long s_ext_extents;
#endif

        /* for buddy allocator */
        struct ext4_group_info ***s_group_info;
        struct inode *s_buddy_cache;
        spinlock_t s_md_lock;
        unsigned short *s_mb_offsets;
        unsigned int *s_mb_maxs;
        unsigned int s_group_info_size;

        /* tunables */
        unsigned long s_stripe;
        unsigned int s_mb_stream_request;
        unsigned int s_mb_max_to_scan;
        unsigned int s_mb_min_to_scan;
        unsigned int s_mb_stats;
        unsigned int s_mb_order2_reqs;
        unsigned int s_mb_group_prealloc;
        unsigned int s_max_dir_size_kb;
        /* where last allocation was done - for stream allocation */
        unsigned long s_mb_last_group;
        unsigned long s_mb_last_start;

        /* stats for buddy allocator */
        atomic_t s_bal_reqs;    /* number of reqs with len > 1 */
        atomic_t s_bal_success; /* we found long enough chunks */
        atomic_t s_bal_allocated;       /* in blocks */
        atomic_t s_bal_ex_scanned;      /* total extents scanned */
        atomic_t s_bal_goals;   /* goal hits */
        atomic_t s_bal_breaks;  /* too long searches */
        atomic_t s_bal_2orders; /* 2^order hits */
        spinlock_t s_bal_lock;
        unsigned long s_mb_buddies_generated;
        unsigned long long s_mb_generation_time;
        atomic_t s_mb_lost_chunks;
        atomic_t s_mb_preallocated;
        atomic_t s_mb_discarded;
        atomic_t s_lock_busy;

        /* locality groups */
        struct ext4_locality_group __percpu *s_locality_groups;

        /* for write statistics */
        unsigned long s_sectors_written_start;
        u64 s_kbytes_written;

        /* the size of zero-out chunk */
        unsigned int s_extent_max_zeroout_kb;

        unsigned int s_log_groups_per_flex;
        struct flex_groups *s_flex_groups;
        ext4_group_t s_flex_groups_allocated;

        /* workqueue for reserved extent conversions (buffered io) */
        struct workqueue_struct *rsv_conversion_wq;

        /* timer for periodic error stats printing */
        struct timer_list s_err_report;

        /* Lazy inode table initialization info */
        struct ext4_li_request *s_li_request;
        /* Wait multiplier for lazy initialization thread */
        unsigned int s_li_wait_mult;

        /* Kernel thread for multiple mount protection */
        struct task_struct *s_mmp_tsk;

        /* record the last minlen when FITRIM is called. */
        atomic_t s_last_trim_minblks;

        /* Reference to checksum algorithm driver via cryptoapi */
        struct crypto_shash *s_chksum_driver;

        /* Precomputed FS UUID checksum for seeding other checksums */
        __u32 s_csum_seed;

        /* Reclaim extents from extent status tree */
        struct shrinker s_es_shrinker;
        struct list_head s_es_list;     /* List of inodes with reclaimable extents */
        long s_es_nr_inode;
        struct ext4_es_stats s_es_stats;
        struct mb2_cache *s_mb_cache;
        spinlock_t s_es_lock ____cacheline_aligned_in_smp;

        /* Ratelimit ext4 messages. */
        struct ratelimit_state s_err_ratelimit_state;
        struct ratelimit_state s_warning_ratelimit_state;
        struct ratelimit_state s_msg_ratelimit_state;
};
```

- 5、ext4的内存inode结构

```c++
// file: fs/ext4/ext4.h
/*
 * fourth extended file system inode data in memory
 */
struct ext4_inode_info {
        __le32  i_data[15];     /* unconverted */
        __u32   i_dtime;
        ext4_fsblk_t    i_file_acl;

        /*
         * i_block_group is the number of the block group which contains
         * this file's inode.  Constant across the lifetime of the inode,
         * it is used for making block allocation decisions - we try to
         * place a file's data blocks near its inode block, and new inodes
         * near to their parent directory's inode.
         */
        ext4_group_t    i_block_group;
        ext4_lblk_t     i_dir_start_lookup;
#if (BITS_PER_LONG < 64)
        unsigned long   i_state_flags;          /* Dynamic state flags */
#endif
        unsigned long   i_flags;

        /*
         * Extended attributes can be read independently of the main file
         * data. Taking i_mutex even when reading would cause contention
         * between readers of EAs and writers of regular file data, so
         * instead we synchronize on xattr_sem when reading or changing
         * EAs.
         */
        struct rw_semaphore xattr_sem;

        struct list_head i_orphan;      /* unlinked but open inodes */

        /* Fast commit related info */

        struct list_head i_fc_list;     /*
                                         * inodes that need fast commit
                                         * protected by sbi->s_fc_lock.
                                         */

        /* Start of lblk range that needs to be committed in this fast commit */
        ext4_lblk_t i_fc_lblk_start;

        /* End of lblk range that needs to be committed in this fast commit */
        ext4_lblk_t i_fc_lblk_len;

        /* Number of ongoing updates on this inode */
        atomic_t  i_fc_updates;

        /* Fast commit wait queue for this inode */
        wait_queue_head_t i_fc_wait;

        /* Protect concurrent accesses on i_fc_lblk_start, i_fc_lblk_len */
        struct mutex i_fc_lock;

                /*
         * i_disksize keeps track of what the inode size is ON DISK, not
         * in memory.  During truncate, i_size is set to the new size by
         * the VFS prior to calling ext4_truncate(), but the filesystem won't
         * set i_disksize to 0 until the truncate is actually under way.
         *
         * The intent is that i_disksize always represents the blocks which
         * are used by this file.  This allows recovery to restart truncate
         * on orphans if we crash during truncate.  We actually write i_disksize
         * into the on-disk inode when writing inodes out, instead of i_size.
         *
         * The only time when i_disksize and i_size may be different is when
         * a truncate is in progress.  The only things which change i_disksize
         * are ext4_get_block (growth) and ext4_truncate (shrinkth).
         */
        loff_t  i_disksize;

        /*
         * i_data_sem is for serialising ext4_truncate() against
         * ext4_getblock().  In the 2.4 ext2 design, great chunks of inode's
         * data tree are chopped off during truncate. We can't do that in
         * ext4 because whenever we perform intermediate commits during
         * truncate, the inode and all the metadata blocks *must* be in a
         * consistent state which allows truncation of the orphans to restart
         * during recovery.  Hence we must fix the get_block-vs-truncate race
         * by other means, so we have i_data_sem.
         */
        struct rw_semaphore i_data_sem;
        /*
         * i_mmap_sem is for serializing page faults with truncate / punch hole
         * operations. We have to make sure that new page cannot be faulted in
         * a section of the inode that is being punched. We cannot easily use
         * i_data_sem for this since we need protection for the whole punch
         * operation and i_data_sem ranks below transaction start so we have
         * to occasionally drop it.
         */
        struct rw_semaphore i_mmap_sem;
        struct inode vfs_inode;
        struct jbd2_inode *jinode;

        spinlock_t i_raw_lock;  /* protects updates to the raw inode */

        /*
         * File creation time. Its function is same as that of
         * struct timespec64 i_{a,c,m}time in the generic inode.
         */
        struct timespec64 i_crtime;

        /* mballoc */
        atomic_t i_prealloc_active;
        struct list_head i_prealloc_list;
        spinlock_t i_prealloc_lock;

        /* extents status tree */
        struct ext4_es_tree i_es_tree;
        rwlock_t i_es_lock;
        struct list_head i_es_list;
        unsigned int i_es_all_nr;       /* protected by i_es_lock */
        unsigned int i_es_shk_nr;       /* protected by i_es_lock */
        ext4_lblk_t i_es_shrink_lblk;   /* Offset where we start searching for
                                           extents to shrink. Protected by
                                           i_es_lock  */

        /* ialloc */
        ext4_group_t    i_last_alloc_group;

        /* allocation reservation info for delalloc */
        /* In case of bigalloc, this refer to clusters rather than blocks */
        unsigned int i_reserved_data_blocks;

        /* pending cluster reservations for bigalloc file systems */
        struct ext4_pending_tree i_pending_tree;

        /* on-disk additional length */
        __u16 i_extra_isize;

        /* Indicate the inline data space. */
        u16 i_inline_off;
        u16 i_inline_size;

#ifdef CONFIG_QUOTA
        /* quota space reservation, managed internally by quota code */
        qsize_t i_reserved_quota;
#endif

        /* Lock protecting lists below */
        spinlock_t i_completed_io_lock;
        /*
         * Completed IOs that need unwritten extents handling and have
         * transaction reserved
         */
        struct list_head i_rsv_conversion_list;
        struct work_struct i_rsv_conversion_work;
        atomic_t i_unwritten; /* Nr. of inflight conversions pending */

        spinlock_t i_block_reservation_lock;

        /*
         * Transactions that contain inode's metadata needed to complete
         * fsync and fdatasync, respectively.
         */
        tid_t i_sync_tid;
        tid_t i_datasync_tid;

#ifdef CONFIG_QUOTA
        struct dquot *i_dquot[MAXQUOTAS];
#endif

        /* Precomputed uuid+inum+igen checksum for seeding inode checksums */
        __u32 i_csum_seed;
        kprojid_t i_projid;
};
```

- 6、ext4磁盘inode结构

```c++
// file: fs/ext4/ext4.h
/*
 * Structure of an inode on the disk
 */
struct ext4_inode { 
        __le16  i_mode;         /* File mode */
        __le16  i_uid;          /* Low 16 bits of Owner Uid */
        __le32  i_size_lo;      /* Size in bytes */
        __le32  i_atime;        /* Access time */
        __le32  i_ctime;        /* Inode Change time */
        __le32  i_mtime;        /* Modification time */
        __le32  i_dtime;        /* Deletion Time */
        __le16  i_gid;          /* Low 16 bits of Group Id */
        __le16  i_links_count;  /* Links count */
        __le32  i_blocks_lo;    /* Blocks count */
        __le32  i_flags;        /* File flags */
        union {      
                struct {
                        __le32  l_i_version;
                } linux1;   
                struct {    
                        __u32  h_i_translator;
                } hurd1;
                struct {
                        __u32  m_i_reserved1;
                } masix1;
        } osd1;                         /* OS dependent 1 */
        __le32  i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
        __le32  i_generation;   /* File version (for NFS) */
        __le32  i_file_acl_lo;  /* File ACL */
        __le32  i_size_high;
        __le32  i_obso_faddr;   /* Obsoleted fragment address */
        union {
                struct {
                        __le16  l_i_blocks_high; /* were l_i_reserved1 */
                        __le16  l_i_file_acl_high;
                        __le16  l_i_uid_high;   /* these 2 fields */
                        __le16  l_i_gid_high;   /* were reserved2[0] */
                        __le16  l_i_checksum_lo;/* crc32c(uuid+inum+inode) LE */
                        __le16  l_i_reserved;
                } linux2;
                struct {
                        __le16  h_i_reserved1;  /* Obsoleted fragment number/size which are removed in ext4 */
                        __u16   h_i_mode_high;
                        __u16   h_i_uid_high;
                        __u16   h_i_gid_high;
                        __u32   h_i_author;
                } hurd2;
                struct {
                        __le16  h_i_reserved1;  /* Obsoleted fragment number/size which are removed in ext4 */
                        __le16  m_i_file_acl_high;
                        __u32   m_i_reserved2[2];
                } masix2;
        } osd2;                         /* OS dependent 2 */
        __le16  i_extra_isize;
        __le16  i_checksum_hi;  /* crc32c(uuid+inum+inode) BE */
        __le32  i_ctime_extra;  /* extra Change time      (nsec << 2 | epoch) */
        __le32  i_mtime_extra;  /* extra Modification time(nsec << 2 | epoch) */
        __le32  i_atime_extra;  /* extra Access time      (nsec << 2 | epoch) */
        __le32  i_crtime;       /* File Creation time */
        __le32  i_crtime_extra; /* extra FileCreationtime (nsec << 2 | epoch) */
        __le32  i_version_hi;   /* high 32 bits for 64-bit version */
        __le32  i_projid;       /* Project ID */
};
```

### 3、初始化
