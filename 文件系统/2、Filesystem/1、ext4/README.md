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

- 2、ext4的superblock**定义**

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
        .put_super      = ext4_put_super,
        .sync_fs        = ext4_sync_fs,
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

### 3、初始化
