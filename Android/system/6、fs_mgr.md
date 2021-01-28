**ref**：
```
https://www.dazhuanlan.com/2019/11/15/5dcd8622cb6a7/
https://www.cnblogs.com/xiaolei-kaiyuan/p/5501104.html
```
---

### 1、概要

### 2、调用流程  

**<font color="red">几个重要的结构体</font>**
```c++
// file ： /android/system/core/fs_mgr/include_fstab/fstab/fstab.h
struct fstab {
    int num_entries;                // 记录fstab.qcom的有效行数
    struct fstab_rec *recs;
    char *fstab_filename;           // /vendor/etc/fstab.qcom
};

// /vendor/etc/fstab.qcom中的一行
struct fstab_rec {
    char *blk_device;           // 块设备路径
    char *mount_point;          // 挂载点
    char *fs_type;              // 文件系统类型
    unsigned long flags;        // mount系统调用需要用到的flag
    char *fs_options;           // mount系统调用需要用到的option
    int fs_mgr_flags;
    char *key_loc;
    char *verity_loc;
    long long length;
    char *label;
    int partnum;
    int swap_prio;
    unsigned int zram_size;
};

// file：android/system/core/fs_mgr/fs_mgr_fstab.cpp
// fs_mgr用来管理分区的标志
struct fs_mgr_flag_values {
    char *key_loc;
    char* key_dir;
    char *verity_loc;
    long long part_length;
    char *label;
    int partnum;
    int swap_prio;
    int max_comp_streams;
    unsigned int zram_size;
    uint64_t reserved_size;
    unsigned int file_contents_mode;
    unsigned int file_names_mode;
    unsigned int erase_blk_size;
    unsigned int logical_blk_size;
};
```

- 1、init进程读取init.target.rc（**android/device/xxx_project/xxx_platform**）中的规则启动对应的服务，其中fs的配置如下  

```bash
on fs
    wait /dev/block/platform/soc/${ro.boot.bootdevice}
    symlink /dev/block/platform/soc/${ro.boot.bootdevice} /dev/block/bootdevice
    mount_all /vendor/etc/fstab.qcom    #解析到这里会调用do_mount_all函数，参数是：/vendor/etc/fstab.qcom
    swapon_all /vendor/etc/fstab.qcom

    # Keeping following partitions outside fstab file. As user may not have
    # these partition flashed on the device. Failure to mount any partition in fstab file
    # results in failure to launch late-start class.

    wait /dev/block/bootdevice/by-name/persist
    mount ext4 /dev/block/bootdevice/by-name/persist /persist noatime nosuid nodev barrier=1
    mkdir /persist/data 0700 system system
    mkdir /persist/bms 0700 root system
    mkdir /persist/account 0700 system system
    mkdir /persist/svm 0700 system system
    restorecon_recursive /persist
```

- 2、do_mount_all执行mount分区操作  

```c++
// file：android/system/core/init/builtins.cpp

/* mount_all <fstab> [ <path> ]* [--<options>]*
   *
   * This function might request a reboot, in which case it will
   * not return.
   */
static int do_mount_all(const std::vector<std::string>& args) {
    std::size_t na = 0;
    bool import_rc = true;
    bool queue_event = true;
    int mount_mode = MOUNT_MODE_DEFAULT;
    const char* fstabfile = args[1].c_str();
    std::size_t path_arg_end = args.size();
    const char* prop_post_fix = "default";

    for (na = args.size() - 1; na > 1; --na) {
        if (args[na] == "--early") {
            path_arg_end = na;
            queue_event = false;
            mount_mode = MOUNT_MODE_EARLY;
            prop_post_fix = "early";
        } else if (args[na] == "--late") {
            path_arg_end = na;
            import_rc = false;
            mount_mode = MOUNT_MODE_LATE;
            prop_post_fix = "late";
        }
    }

    std::string prop_name = "ro.boottime.init.mount_all."s + prop_post_fix;
    android::base::Timer t;
    // mount_fstab会去执行挂载分区的动作
    int ret =  mount_fstab(fstabfile, mount_mode);
    property_set(prop_name, std::to_string(t.duration().count()));

    if (import_rc) {
        /* Paths of .rc files are specified at the 2nd argument and beyond */
        import_late(args, 2, path_arg_end);
    }

    if (queue_event) {
        /* queue_fs_event will queue event based on mount_fstab return code
        * and return processed return code*/
        ret = queue_fs_event(ret);
    }

    return ret;
}
```

- 3、下面核心就是**mount_fstab**函数

```c++
// file ： android/system/core/init/builtins.cpp
static int mount_fstab(const char* fstabfile, int mount_mode) {
    int ret = -1;

    /*
    * Call fs_mgr_mount_all() to mount all filesystems.  We fork(2) 
    * do the call in the child to provide protection to the main init
    * process if anything goes wrong (crash or memory leak), and wait for
    * the child to finish in the parent.
    */
    pid_t pid = fork();
    if (pid > 0) {
        /* Parent.  Wait for the child to return */
        int status;
        int wp_ret = TEMP_FAILURE_RETRY(waitpid(pid, &status, 0));
        if (wp_ret == -1) {
            // Unexpected error code. We will continue anyway.
            PLOG(WARNING) << "waitpid failed";
        }

        if (WIFEXITED(status)) {
            ret = WEXITSTATUS(status);
        } else {
            ret = -1;
        }
    } else if (pid == 0) { // 子进程，fork子进程是为了防止init进程die
        /* child, call fs_mgr_mount_all() */

        // So we can always see what fs_mgr_mount_all() does.
        // Only needed if someone explicitly changes the default log level in their init.rc.
        android::base::ScopedLogSeverity info(android::base::INFO);

        // 读取/vendor/etc/fstab.qcom获取挂载参数
        struct fstab* fstab = fs_mgr_read_fstab(fstabfile);
        // 执行mount操作
        int child_ret = fs_mgr_mount_all(fstab, mount_mode);
        fs_mgr_free_fstab(fstab);
        if (child_ret == -1) {
            PLOG(ERROR) << "fs_mgr_mount_all returned an error";
        }
        _exit(child_ret);
    } else {
        /* fork failed, return an error */
        return -1;
    }
    return ret;
}
```

- 3、**fs_mgr_read_fstab**函数负责读取/vendor/etc/fstab.qcom解析配置,返回fstab结构  

```c++
// file ： android/system/core/fs_mgr/fs_mgr_fstab.cpp
struct fstab *fs_mgr_read_fstab(const char *fstab_path)
{
    FILE *fstab_file;
    struct fstab *fstab;

    fstab_file = fopen(fstab_path, "r");
    if (!fstab_file) {
        PERROR << __FUNCTION__<< "(): cannot open file: '" << fstab_path << "'";
        return nullptr;
    }

    // 遍历/vendor/etc/fstab.qcom文件
    fstab = fs_mgr_read_fstab_file(fstab_file);
    if (fstab) {
        fstab->fstab_filename = strdup(fstab_path);
    } else {
        LERROR << __FUNCTION__ << "(): failed to load fstab from : '" << fstab_path << "'";
    }

    fclose(fstab_file);
    return fstab;
}

// 返回的fstab结构包含了解析/vendor/etc/fstab.qcom的所有行
static struct fstab *fs_mgr_read_fstab_file(FILE *fstab_file)
{
    int cnt, entries;
    ssize_t len;
    size_t alloc_len = 0;
    char *line = NULL;
    const char *delim = " \t";      // 制表符
    char *save_ptr, *p;
    struct fstab *fstab = NULL;
    struct fs_mgr_flag_values flag_vals;
#define FS_OPTIONS_LEN 1024
    char tmp_fs_options[FS_OPTIONS_LEN];

    entries = 0;
    // 这里只是获取fstab.qcom中有效行的个数为了后面给fstab->recs分配内存
    while ((len = getline(&line, &alloc_len, fstab_file)) != -1) {  // 读取一行
        /* if the last character is a newline, shorten the string by 1 byte */
        if (line[len - 1] == '\n') {
            line[len - 1] = '\0';
        }
        /* Skip any leading whitespace */
        p = line;
        while (isspace(*p)) {
            p++;
        }
        /* ignore comments or empty lines */
        if (*p == '#' || *p == '\0')
            continue;
        entries++;          // fstab.qcom有效行数++
    }

    if (!entries) {     // fstab.qcom有效行不存在
        LERROR << "No entries found in fstab";
        goto err;
    }

    /* Allocate and init the fstab structure */
    fstab = static_cast<struct fstab *>(calloc(1, sizeof(struct fstab)));
    // fstab.qcom的有效行数赋值给fastab->num_entries
    fstab->num_entries = entries;
    // 每一行对应一个fstab_rec结构
    fstab->recs = static_cast<struct fstab_rec *>(
        calloc(fstab->num_entries, sizeof(struct fstab_rec)));
    // 偏移到fstab.qcom的开头，因为上面getline使得文件指针偏移了
    fseek(fstab_file, 0, SEEK_SET);

    cnt = 0;
    // 重新解析fstab.qcom
    while ((len = getline(&line, &alloc_len, fstab_file)) != -1) {
        /* if the last character is a newline, shorten the string by 1 byte */
        if (line[len - 1] == '\n') {
            line[len - 1] = '\0';
        }

        /* Skip any leading whitespace */
        // p指向line包含的字符串（带'\0'）
        p = line;
        // 找到line中第一个不为空的字符
        while (isspace(*p)) {
            p++;
        }

        // 跳过注释和空行
        /* ignore comments or empty lines */
        if (*p == '#' || *p == '\0')
            continue;

        /* If a non-comment entry is greater than the size we allocated, give an
        * error and quit.  This can happen in the unlikely case the file changes
        * between the two reads.
        */
        if (cnt >= entries) {
            LERROR << "Tried to process more entries than counted";
            break;
        }

        // 1、以制表符分割一行，也就是取一行中的第一列，即块设备路径，如：/dev/block/bootdevice/by-name/system
        if (!(p = strtok_r(line, delim, &save_ptr))) {
            LERROR << "Error parsing mount source";
            goto err;
        }
        fstab->recs[cnt].blk_device = strdup(p);

        // 2、取第二列，挂载点
        if (!(p = strtok_r(NULL, delim, &save_ptr))) {
            LERROR << "Error parsing mount_point";
            goto err;
        }
        fstab->recs[cnt].mount_point = strdup(p);

        // 3、取第三列，文件系统类型
        if (!(p = strtok_r(NULL, delim, &save_ptr))) {
            LERROR << "Error parsing fs_type";
            goto err;
        }
        fstab->recs[cnt].fs_type = strdup(p);

        // 4、取第四列挂载参数，userdata分区：noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard
        if (!(p = strtok_r(NULL, delim, &save_ptr))) {
            LERROR << "Error parsing mount_flags";
            goto err;
        }
        tmp_fs_options[0] = '\0';
        fstab->recs[cnt].flags = parse_flags(p, mount_flags, NULL,
                                        tmp_fs_options, FS_OPTIONS_LEN);
        /*
            static struct flag_list mount_flags[] = {
                { "noatime",    MS_NOATIME },
                { "noexec",     MS_NOEXEC },
                { "nosuid",     MS_NOSUID },
                { "nodev",      MS_NODEV },
                { "nodiratime", MS_NODIRATIME },
                { "ro",         MS_RDONLY },
                { "rw",         0 },
                { "remount",    MS_REMOUNT },
                { "bind",       MS_BIND },
                { "rec",        MS_REC },
                { "unbindable", MS_UNBINDABLE },
                { "private",    MS_PRIVATE },
                { "slave",      MS_SLAVE },
                { "shared",     MS_SHARED },
                { "defaults",   0 },
                { 0,            0 },
            };
        */

        /* fs_options are optional */
        if (tmp_fs_options[0]) {
            fstab->recs[cnt].fs_options = strdup(tmp_fs_options);
        } else {
            fstab->recs[cnt].fs_options = NULL;
        }

        // 5、取第五列fs_mgr_flags，userdata是：wait,encryptable=footer,reservedsize=192M
        if (!(p = strtok_r(NULL, delim, &save_ptr))) {
            LERROR << "Error parsing fs_mgr_options";
            goto err;
        }
        fstab->recs[cnt].fs_mgr_flags = parse_flags(p, fs_mgr_flags,
                                                    &flag_vals, NULL, 0);
        /*
            static struct flag_list fs_mgr_flags[] = {
                {"wait", MF_WAIT},
                {"check", MF_CHECK},
                {"encryptable=", MF_CRYPT},
                {"forceencrypt=", MF_FORCECRYPT},
                {"fileencryption=", MF_FILEENCRYPTION},
                {"forcefdeorfbe=", MF_FORCEFDEORFBE},
                {"keydirectory=", MF_KEYDIRECTORY},
                {"nonremovable", MF_NONREMOVABLE},
                {"voldmanaged=", MF_VOLDMANAGED},
                {"length=", MF_LENGTH},
                {"recoveryonly", MF_RECOVERYONLY},
                {"swapprio=", MF_SWAPPRIO},
                {"zramsize=", MF_ZRAMSIZE},
                {"max_comp_streams=", MF_MAX_COMP_STREAMS},
                {"verifyatboot", MF_VERIFYATBOOT},
                {"verify", MF_VERIFY},
                {"avb", MF_AVB},
                {"noemulatedsd", MF_NOEMULATEDSD},
                {"notrim", MF_NOTRIM},
                {"formattable", MF_FORMATTABLE},
                {"slotselect", MF_SLOTSELECT},
                {"nofail", MF_NOFAIL},
                {"latemount", MF_LATEMOUNT},
                {"reservedsize=", MF_RESERVEDSIZE},
                {"quota", MF_QUOTA},
                {"eraseblk=", MF_ERASEBLKSIZE},
                {"logicalblk=", MF_LOGICALBLKSIZE},
                {"defaults", 0},
                {0, 0},
            };
        */
        fstab->recs[cnt].key_loc = flag_vals.key_loc;
        fstab->recs[cnt].key_dir = flag_vals.key_dir;
        fstab->recs[cnt].verity_loc = flag_vals.verity_loc;
        fstab->recs[cnt].length = flag_vals.part_length;
        fstab->recs[cnt].label = flag_vals.label;
        fstab->recs[cnt].partnum = flag_vals.partnum;
        fstab->recs[cnt].swap_prio = flag_vals.swap_prio;
        fstab->recs[cnt].max_comp_streams = flag_vals.max_comp_streams;
        fstab->recs[cnt].zram_size = flag_vals.zram_size;
        fstab->recs[cnt].reserved_size = flag_vals.reserved_size;
        fstab->recs[cnt].file_contents_mode = flag_vals.file_contents_mode;
        fstab->recs[cnt].file_names_mode = flag_vals.file_names_mode;
        fstab->recs[cnt].erase_blk_size = flag_vals.erase_blk_size;
        fstab->recs[cnt].logical_blk_size = flag_vals.logical_blk_size;
        cnt++;
    }
    /* If an A/B partition, modify block device to be the real block device */
    if (!fs_mgr_update_for_slotselect(fstab)) {
        LERROR << "Error updating for slotselect";
        goto err;
    }
    free(line);
    return fstab;

err:
    free(line);
    if (fstab)
        fs_mgr_free_fstab(fstab);
    return NULL;
}
``` 
**file：/vendor/etc/fstab.qcom**
```bash
# Android fstab file.
# The filesystem that contains the filesystem checker binary (typically /system) cannot
# specify MF_CHECK, and must come before any filesystems that do specify MF_CHECK

#TODO: Add 'check' as fs_mgr_flags with data partition.
# Currently we dont have e2fsck compiled. So fs check would failed.

#<src>                                  <mnt_point>       <type>  <mnt_flags and options>                     <fs_mgr_flags>
/dev/block/bootdevice/by-name/system    /                 ext4    ro,barrier=1,discard                                wait,slotselect,verify
/dev/block/bootdevice/by-name/ota_cache /ota_chj/otacache             ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer
/dev/block/bootdevice/by-name/otaback_a /ota_chj/otabak_a             ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer
/dev/block/bootdevice/by-name/otaback_b /ota_chj/otabak_b            ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer
/dev/block/bootdevice/by-name/tts       /tts             ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer
/dev/block/bootdevice/by-name/can_data  /can_data             ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer
/dev/block/bootdevice/by-name/diag_data /diag_data             ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer
/dev/block/bootdevice/by-name/avm_calibration  /avm_calibration            ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer
/dev/block/bootdevice/by-name/video_data /video_data             ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer
/dev/block/bootdevice/by-name/log_data /log             ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer
/dev/block/bootdevice/by-name/track_data /track_data             ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer
/dev/block/bootdevice/by-name/userdata  /data             ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer,reservedsize=192M
/devices/soc/74a4900.sdhci/mmc_host*    /storage/sdcard1  vfat    nosuid,nodev                                        wait,voldmanaged=sdcard1:auto,encryptable=footer
#xhci-hcd.0.auto/usb* used for USB drive mounting on automotive
/devices/*/xhci-hcd.*.auto/usb*         auto              auto    defaults                                            voldmanaged=usb:auto
/dev/block/bootdevice/by-name/misc      /misc             emmc    defaults                                            defaults
/dev/block/bootdevice/by-name/dsp       /dsp              ext4    ro,nosuid,nodev,barrier=1                           wait,slotselect
/dev/block/bootdevice/by-name/modem     /firmware         vfat    ro,shortname=lower,uid=1000,gid=1000,dmask=227,fmask=337,context=u:object_r:firmware_file:s0  wait,slotselect
/dev/block/bootdevice/by-name/bluetooth /bt_firmware      vfat    ro,shortname=lower,uid=1002,gid=3002,dmask=227,fmask=337,context=u:object_r:bt_firmware_file:s0  wait,slotselect
```

- 4、**fs_mgr_mount_all** 真正的执行mount动作  

```c++
//file：android/system/core/fs_mgr/fs_mgr.cpp
/* When multiple fstab records share the same mount_point, it will
   * try to mount each one in turn, and ignore any duplicates after a
   * first successful mount.
   * Returns -1 on error, and  FS_MGR_MNTALL_* otherwise.
   */
int fs_mgr_mount_all(struct fstab *fstab, int mount_mode)
{
    int i = 0;
    int encryptable = FS_MGR_MNTALL_DEV_NOT_ENCRYPTABLE;    // 加密分区标记
    int error_count = 0;
    int mret = -1;
    int mount_errno = 0;
    int attempted_idx = -1;
    FsManagerAvbUniquePtr avb_handle(nullptr);
    char propbuf[PROPERTY_VALUE_MAX];
    bool is_ffbm = false;

    if (!fstab) {
        return FS_MGR_MNTALL_FAIL;
    }

    /**get boot mode*/
    property_get("ro.bootmode", propbuf, "");
    if ((strncmp(propbuf, "ffbm-00", 7) == 0) || (strncmp(propbuf, "ffbm-01", 7) == 0))
        is_ffbm = true;

    // 遍历/vendor/etc/fstab.com中的每一行
    for (i = 0; i < fstab->num_entries; i++) {
        /* Skip userdata partition in ffbm mode */
        if (is_ffbm && !strcmp(fstab->recs[i].mount_point, "/data")){
            continue;
        }

        /* Don't mount entries that are managed by vold or not for the mount mode*/
        if ((fstab->recs[i].fs_mgr_flags & (MF_VOLDMANAGED | MF_RECOVERYONLY)) ||
            ((mount_mode == MOUNT_MODE_LATE) && !fs_mgr_is_latemount(&fstab->recs[i])) ||
            ((mount_mode == MOUNT_MODE_EARLY) && fs_mgr_is_latemount(&fstab->recs[i]))) {
            continue;
        }

        /* Skip swap and raw partition entries such as boot, recovery, etc */
        if (!strcmp(fstab->recs[i].fs_type, "swap") ||
            !strcmp(fstab->recs[i].fs_type, "emmc") ||
            !strcmp(fstab->recs[i].fs_type, "mtd")) {
            continue;
        }

        /* Skip mounting the root partition, as it will already have been mounted */
        /*
            不mount root分区(system.img:/dev/block/mmcblk0p18）
            在kernel启动时已经挂载mmcblk0p18为rootfs了，下面时cmdline中的配置：
                init=/init root=/dev/dm-0 dm="system none ro,0 1 android-verity /dev/mmcblk0p18"
        */
        if (!strcmp(fstab->recs[i].mount_point, "/")) {
            if ((fstab->recs[i].fs_mgr_flags & MS_RDONLY) != 0) {
                fs_mgr_set_blk_ro(fstab->recs[i].blk_device);
            }
            continue;
        }

        /* Translate LABEL= file system labels into block devices */
        if (is_extfs(fstab->recs[i].fs_type)) { // extx文件文件系统
            int tret = translate_ext_labels(&fstab->recs[i]);
            if (tret < 0) {
                LERROR << "Could not translate label to block device";
                continue;
            }
        }

        // 当有"wait"关键字时，让系统sleep一会
        if (fstab->recs[i].fs_mgr_flags & MF_WAIT &&
            !fs_mgr_wait_for_file(fstab->recs[i].blk_device, 20s)) {
            LERROR << "Skipping '" << fstab->recs[i].blk_device << "' during mount_all";
            continue;
        }

        if (fstab->recs[i].fs_mgr_flags & MF_AVB) {
            if (!avb_handle) {
                avb_handle = FsManagerAvbHandle::Open(*fstab);
                if (!avb_handle) {
                    LERROR << "Failed to open FsManagerAvbHandle";
                    return FS_MGR_MNTALL_FAIL;
                }
            }
            if (avb_handle->SetUpAvbHashtree(&fstab->recs[i], true /* wait_for_verity_dev */) ==
                SetUpAvbHashtreeResult::kFail) {
                LERROR << "Failed to set up AVB on partition: "
                        << fstab->recs[i].mount_point << ", skipping!";
                /* Skips mounting the device. */
                continue;
            }
        } else if ((fstab->recs[i].fs_mgr_flags & MF_VERIFY) && is_device_secure()) {
            int rc = fs_mgr_setup_verity(&fstab->recs[i], true);
            if (__android_log_is_debuggable() &&
                    (rc == FS_MGR_SETUP_VERITY_DISABLED ||
                    rc == FS_MGR_SETUP_VERITY_SKIPPED)) {
                LINFO << "Verity disabled";
            } else if (rc != FS_MGR_SETUP_VERITY_SUCCESS) {
                LERROR << "Could not set up verified partition, skipping!";
                continue;
            }
        }

        int last_idx_inspected;
        int top_idx = i;

        /*
            功能：挂载
                fstab                   ：代码/vendor/etc/fstab.qcom文件解析结果
                i                       ：每一行，代表一个分区
                last_idx_inspected      ：
                attempted_idx           ：挂载失败设置为i
        */
        mret = mount_with_alternatives(fstab, i, &last_idx_inspected, &attempted_idx);
        i = last_idx_inspected;
        mount_errno = errno;

        /* Deal with encryptability. */
        if (!mret) {
            int status = handle_encryptable(&fstab->recs[attempted_idx]);

            if (status == FS_MGR_MNTALL_FAIL) {
                /* Fatal error - no point continuing */
                return status;
            }

            if (status != FS_MGR_MNTALL_DEV_NOT_ENCRYPTABLE) {
                if (encryptable != FS_MGR_MNTALL_DEV_NOT_ENCRYPTABLE) {
                    // Log and continue
                    /*
                        log9：userdata分区检测最后一行日志
                            init: [libfs_mgr]Only one encryptable/encrypted partition supported
                    */
                    LERROR << "Only one encryptable/encrypted partition supported";
                }
                encryptable = status;
            }

            /* Success!  Go get the next one */
            continue;
        }

        bool wiped = partition_wiped(fstab->recs[top_idx].blk_device);
        bool crypt_footer = false;
        if (mret && mount_errno != EBUSY && mount_errno != EACCES &&
            fs_mgr_is_formattable(&fstab->recs[top_idx]) && wiped) {
            /* top_idx and attempted_idx point at the same partition, but sometimes
            * at two different lines in the fstab.  Use the top one for formatting
            * as that is the preferred one.
            */
            LERROR << __FUNCTION__ << "(): " << fstab->recs[top_idx].blk_device
                    << " is wiped and " << fstab->recs[top_idx].mount_point
                    << " " << fstab->recs[top_idx].fs_type
                    << " is formattable. Format it.";
            if (fs_mgr_is_encryptable(&fstab->recs[top_idx]) &&
                strcmp(fstab->recs[top_idx].key_loc, KEY_IN_FOOTER)) {
                int fd = open(fstab->recs[top_idx].key_loc, O_WRONLY);
                if (fd >= 0) {
                    LINFO << __FUNCTION__ << "(): also wipe "
                        << fstab->recs[top_idx].key_loc;
                    wipe_block_device(fd, get_file_size(fd));
                    close(fd);
                } else {
                    PERROR << __FUNCTION__ << "(): "
                            << fstab->recs[top_idx].key_loc << " wouldn't open";
                }
            } else if (fs_mgr_is_encryptable(&fstab->recs[top_idx]) &&
                !strcmp(fstab->recs[top_idx].key_loc, KEY_IN_FOOTER)) {
                crypt_footer = true;
            }
            if (fs_mgr_do_format(&fstab->recs[top_idx], crypt_footer) == 0) {
                /* Let's replay the mount actions. */
                i = top_idx - 1;
                continue;
            } else {
                LERROR << __FUNCTION__ << "(): Format failed. "
                        << "Suggest recovery...";
                encryptable = FS_MGR_MNTALL_DEV_NEEDS_RECOVERY;
                continue;
            }
        }

        /* mount(2) returned an error, handle the encryptable/formattable case */
        if (mret && mount_errno != EBUSY && mount_errno != EACCES &&
            fs_mgr_is_encryptable(&fstab->recs[attempted_idx])) {
            if (wiped) {
                LERROR << __FUNCTION__ << "(): "
                        << fstab->recs[attempted_idx].blk_device
                        << " is wiped and "
                        << fstab->recs[attempted_idx].mount_point << " "
                        << fstab->recs[attempted_idx].fs_type
                        << " is encryptable. Suggest recovery...";
                encryptable = FS_MGR_MNTALL_DEV_NEEDS_RECOVERY;
                continue;
            } else {
                /* Need to mount a tmpfs at this mountpoint for now, and set
                * properties that vold will query later for decrypting
                */
                LERROR << __FUNCTION__ << "(): possibly an encryptable blkdev "
                        << fstab->recs[attempted_idx].blk_device
                        << " for mount " << fstab->recs[attempted_idx].mount_point
                        << " type " << fstab->recs[attempted_idx].fs_type;
                if (fs_mgr_do_tmpfs_mount(fstab->recs[attempted_idx].mount_point) < 0) {
                    ++error_count;
                    continue;
                }
            }
            encryptable = FS_MGR_MNTALL_DEV_MIGHT_BE_ENCRYPTED;
        } else if (mret && mount_errno != EBUSY && mount_errno != EACCES &&
                    should_use_metadata_encryption(&fstab->recs[attempted_idx])) {
            encryptable = FS_MGR_MNTALL_DEV_IS_METADATA_ENCRYPTED;
        } else {
            // fs_options might be null so we cannot use PERROR << directly.
            // Use StringPrintf to output "(null)" instead.
            if (fs_mgr_is_nofail(&fstab->recs[attempted_idx])) {
                PERROR << android::base::StringPrintf(
                    "Ignoring failure to mount an un-encryptable or wiped "
                    "partition on %s at %s options: %s",
                    fstab->recs[attempted_idx].blk_device, fstab->recs[attempted_idx].mount_point,
                    fstab->recs[attempted_idx].fs_options);
            } else {
                PERROR << android::base::StringPrintf(
                    "Failed to mount an un-encryptable or wiped partition "
                    "on %s at %s options: %s",
                    fstab->recs[attempted_idx].blk_device, fstab->recs[attempted_idx].mount_point,
                    fstab->recs[attempted_idx].fs_options);
                ++error_count;
            }
            continue;
        }
    }

    if (error_count) {
        return FS_MGR_MNTALL_FAIL;
    } else {
        return encryptable;
    }
}

/*
* Tries to mount any of the consecutive fstab entries that match
* the mountpoint of the one given by fstab->recs[start_idx].
*
* end_idx: On return, will be the last rec that was looked at.
* attempted_idx: On return, will indicate which fstab rec
*     succeeded. In case of failure, it will be the start_idx.
* Returns
*   -1 on failure with errno set to match the 1st mount failure.
*   0 on success.
*/
static int mount_with_alternatives(struct fstab *fstab, int start_idx, int *end_idx, int *attempted_idx)
{
    int i;
    int mount_errno = 0;
    int mounted = 0;

    if (!end_idx || !attempted_idx || start_idx >= fstab->num_entries) {
    errno = EINVAL;
    if (end_idx) *end_idx = start_idx;
    if (attempted_idx) *attempted_idx = start_idx;
    return -1;
    }

    /* Hunt down an fstab entry for the same mount point that might succeed */
    for (i = start_idx;
        /* We required that fstab entries for the same mountpoint be consecutive */
        i < fstab->num_entries && !strcmp(fstab->recs[start_idx].mount_point, fstab->recs[i].mount_point);
        i++) {
            /*
            * Don't try to mount/encrypt the same mount point again.
            * Deal with alternate entries for the same point which are required to be all following
            * each other.
            */
            if (mounted) {  // 没走到
                LERROR << __FUNCTION__ << "(): skipping fstab dup mountpoint="
                        << fstab->recs[i].mount_point << " rec[" << i
                        << "].fs_type=" << fstab->recs[i].fs_type
                        << " already mounted as "
                        << fstab->recs[*attempted_idx].fs_type;
                continue;
            }

            // 从块设备所在分区读取superblock并解析，ext4分区的superblock在偏移1024的字节处
            int fs_stat = prepare_fs_for_mount(fstab->recs[i].blk_device, &fstab->recs[i]);
            if (fs_stat & FS_STAT_EXT4_INVALID_MAGIC) {
                // ext4_super_block->magic不匹配，跳过
                LERROR << __FUNCTION__ << "(): skipping mount, invalid ext4, mountpoint="
                        << fstab->recs[i].mount_point << " rec[" << i
                        << "].fs_type=" << fstab->recs[i].fs_type;
                mount_errno = EINVAL;  // continue bootup for FDE
                continue;
            }

            int retry_count = 2;    // 最多尝试mount两次
            while (retry_count-- > 0) {
                /* 
                    之前分区出错的话，已经执行过一次mount->umount操作，下面的mount会打印如下log
                    挂载参数：
                        /dev/block/bootdevice/by-name/userdata  /data    ext4    noatime,nosuid,nodev,barrier=1,noauto_da_alloc,discard  wait,encryptable=footer,reservedsize=192M
                        reservedsize=192M：预留192M给日志系统使用
                        wait：挂载前会sleep
                    log7：
                        EXT4-fs (mmcblk0p65): mounted filesystem with ordered data mode. Opts: barrier=1,noauto_da_alloc,discard
                */
                if (!__mount(fstab->recs[i].blk_device, fstab->recs[i].mount_point,
                            &fstab->recs[i])) { // 挂载成功
                    *attempted_idx = i;
                    mounted = 1;
                    if (i != start_idx) {
                        LERROR << __FUNCTION__ << "(): Mounted " << fstab->recs[i].blk_device
                                << " on " << fstab->recs[i].mount_point
                                << " with fs_type=" << fstab->recs[i].fs_type << " instead of "
                                << fstab->recs[start_idx].fs_type;
                    }
                    fs_stat &= ~FS_STAT_FULL_MOUNT_FAILED;
                    mount_errno = 0;
                    break;
                } else {       // 挂载失败，重新e2fsck，再挂载
                    if (retry_count <= 0) break;  // run check_fs only once
                    fs_stat |= FS_STAT_FULL_MOUNT_FAILED;
                    /* back up the first errno for crypto decisions */
                    if (mount_errno == 0) {
                        mount_errno = errno;
                    }
                    // retry after fsck
                    check_fs(fstab->recs[i].blk_device, fstab->recs[i].fs_type,
                            fstab->recs[i].mount_point, &fs_stat);
                }
            }
            log_fs_stat(fstab->recs[i].blk_device, fs_stat);
    }

    /* Adjust i for the case where it was still withing the recs[] */
    if (i < fstab->num_entries) --i;

    *end_idx = i;
    if (!mounted) {
        *attempted_idx = start_idx;
        errno = mount_errno;
        return -1;
    }
    return 0;
}



// Prepare the filesystem on the given block device to be mounted.
//
// If the "check" option was given in the fstab record, or it seems that the
// filesystem was uncleanly shut down, we'll run fsck on the filesystem.
//
// If needed, we'll also enable (or disable) filesystem features as specified by
// the fstab record.
//
/*
    blk_device：块设备路径，userdata是：/dev/block/bootdevice/by-name/userdata
    rec：代表解析后的fstab.qcom中的一行
*/
static int prepare_fs_for_mount(const char* blk_device, const struct fstab_rec* rec) {
    int fs_stat = 0;

    if (is_extfs(rec->fs_type)) {
        struct ext4_super_block sb;

        // 读取ext4的superblock
        if (read_ext4_superblock(blk_device, &sb, &fs_stat)) {  // 读取成功
            /*
                log2：
                    init: [libfs_mgr]Filesystem on /dev/block/bootdevice/by-name/userdata was not cleanly shutdown; state flags: 0x1, incompat feature flags: 0x2c6
                        0x01:代表clean
                        0x2c6：0x2c6的第3位为1代表EXT4_FEATURE_INCOMPAT_RECOVER被置位，该位置位地方？？？

                #define EXT4_FEATURE_INCOMPAT_RECOVER 0x0004 ？
            /*
                dumpe2fs userdata分区结果：
                    sb.s_state：clean
                        #define EXT4_VALID_FS 0x0001
                        #define EXT4_ERROR_FS 0x0002
                        #define EXT4_ORPHAN_FS 0x0004
                        void print_fs_state (FILE * f, unsigned short state)
                        {
                            if (state & EXT4_VALID_FS)
                                fprintf (f, " clean");
                            else
                                fprintf (f, " not clean");
                            if (state & EXT4_ERROR_FS)
                                fprintf (f, " with errors");
                        }
            */
            if ((sb.s_feature_incompat & EXT4_FEATURE_INCOMPAT_RECOVER) != 0 ||
                (sb.s_state & EXT4_VALID_FS) == 0) {
                LINFO << "Filesystem on " << blk_device << " was not cleanly shutdown; "
                    << "state flags: 0x" << std::hex << sb.s_state << ", "
                    << "incompat feature flags: 0x" << std::hex << sb.s_feature_incompat;
                fs_stat |= FS_STAT_UNCLEAN_SHUTDOWN;
            }

            // Note: quotas should be enabled before running fsck.
            // userdata分区没有开启quota
            tune_quota(blk_device, rec, &sb, &fs_stat);
        } else {    
            // 读取失败，直接返回
            return fs_stat;
        }
    }

    if ((rec->fs_mgr_flags & MF_CHECK) ||
        (fs_stat & (FS_STAT_UNCLEAN_SHUTDOWN | FS_STAT_QUOTA_ENABLED))) { // userdata分区come in，FS_STAT_UNCLEAN_SHUTDOWN被置位
        check_fs(blk_device, rec->fs_type, rec->mount_point, &fs_stat);
    }

    // userdata分区设置了<wait,encryptable=footer,reservedsize=192M>，下面会进去
    if (is_extfs(rec->fs_type) && (rec->fs_mgr_flags & (MF_RESERVEDSIZE | MF_FILEENCRYPTION))) { 
        struct ext4_super_block sb;

        /*
            上面如果ext4文件系统发生了《was not cleanly shutdown》，会进入fs_check执行mount->umount->e2fsck修复，
            然后再次读取sb，第二次在read_ext4_superblock的打印如下：
            log6：没有再打印《was not cleanly shutdown》
                init: [libfs_mgr]superblock s_max_mnt_count:65535,/dev/block/bootdevice/by-name/userdata
        */
        if (read_ext4_superblock(blk_device, &sb, &fs_stat)) {
            // 下面两个函数都在开始返回了
            tune_reserved_size(blk_device, rec, &sb, &fs_stat);
            tune_encrypt(blk_device, rec, &sb, &fs_stat);
        }
    }

    return fs_stat;
}

// Read the primary superblock from an ext4 filesystem.  On failure return
// false.  If it's not an ext4 filesystem, also set FS_STAT_EXT4_INVALID_MAGIC.
static bool read_ext4_superblock(const char* blk_device, struct ext4_super_block* sb, int* fs_stat) {
    // 以只读的方式打开块设备文件
    android::base::unique_fd fd(TEMP_FAILURE_RETRY(open(blk_device, O_RDONLY | O_CLOEXEC)));
    if (fd < 0) {
        PERROR << "Failed to open '" << blk_device << "'";
        return false;
    }

    // 读取的大小是sizeof(*sb)，offset是1024，ext4开头的1024如果是启动分区，则是启动代码，否则全0
    if (pread(fd, sb, sizeof(*sb), 1024) != sizeof(*sb)) {
        PERROR << "Can't read '" << blk_device << "' superblock";
        return false;
    }

    // ext4文件系统有专门的magic，如果不对，认为不是ext4文件系统
    if (sb->s_magic != EXT4_SUPER_MAGIC) {
        LINFO << "Invalid ext4 magic:0x" << std::hex << sb->s_magic << " "
            << "on '" << blk_device << "'";
        // not a valid fs, tune2fs, fsck, and mount  will all fail.
        *fs_stat |= FS_STAT_EXT4_INVALID_MAGIC;
        return false;
    }

    // fs_stat置位FS_STAT_IS_EXT4
    *fs_stat |= FS_STAT_IS_EXT4;
    // log1：
    //     init: [libfs_mgr]superblock s_max_mnt_count:65535,/dev/block/bootdevice/by-name/userdata
    LINFO << "superblock s_max_mnt_count:" << sb->s_max_mnt_count << "," << blk_device;
    if (sb->s_max_mnt_count == 0xffff) {  // -1 (int16) in ext2, but uint16 in ext4
        *fs_stat |= FS_STAT_NEW_IMAGE_VERSION;
    }
    return true;
}

static void check_fs(const char *blk_device, char *fs_type, char *target, int *fs_stat)
{
    int status;
    int ret;
    long tmpmnt_flags = MS_NOATIME | MS_NOEXEC | MS_NOSUID;
    char tmpmnt_opts[64] = "errors=remount-ro";
    const char* e2fsck_argv[] = {E2FSCK_BIN, "-y", blk_device};
    const char* e2fsck_forced_argv[] = {E2FSCK_BIN, "-f", "-y", blk_device};

    /* Check for the types of filesystems we know how to check */
    if (is_extfs(fs_type)) {
        if (*fs_stat & FS_STAT_EXT4_INVALID_MAGIC) {  // will fail, so do not try
            return;
        }
        /*
        * First try to mount and unmount the filesystem.  We do this because
        * the kernel is more efficient than e2fsck in running the journal and
        * processing orphaned inodes, and on at least one device with a
        * performance issue in the emmc firmware, it can take e2fsck 2.5 minutes
        * to do what the kernel does in about a second.
        *
        * After mounting and unmounting the filesystem, run e2fsck, and if an
        * error is recorded in the filesystem superblock, e2fsck will do a full
        * check.  Otherwise, it does nothing.  If the kernel cannot mount the
        * filesytsem due to an error, e2fsck is still run to do a full check
        * fix the filesystem.
        */
        if (!(*fs_stat & FS_STAT_FULL_MOUNT_FAILED)) {  // already tried if full mount failed
            errno = 0;
            if (!strcmp(fs_type, "ext4")) {
                // This option is only valid with ext4
                strlcat(tmpmnt_opts, ",nomblk_io_submit", sizeof(tmpmnt_opts));
            }
            /*
                userdata分区第一次挂载的日志：
                    EXT4-fs (mmcblk0p65): Ignoring removed nomblk_io_submit option
                    EXT4-fs (mmcblk0p65): 15 orphan inodes deleted
                    EXT4-fs (mmcblk0p65): recovery complete
                    EXT4-fs (mmcblk0p65): mounted filesystem with ordered data mode. Opts: errors=remount-ro,nomblk_io_submit
            */
            ret = mount(blk_device, target, fs_type, tmpmnt_flags, tmpmnt_opts);
            /*
                log3：
                    init: [libfs_mgr]check_fs(): mount(/dev/block/bootdevice/by-name/userdata,/data,ext4)=0: Success
                    说明userdata分区挂载成功
            */
            PINFO << __FUNCTION__ << "(): mount(" << blk_device << "," << target << "," << fs_type
                << ")=" << ret;
            if (!ret) { // mount成功再umount，便于e2fsck能fix superblock problem
                bool umounted = false;
                int retry_count = 5;
                while (retry_count-- > 0) {
                    umounted = umount(target) == 0;
                    if (umounted) {
                        /*
                            log4：卸载成功
                                init: [libfs_mgr]check_fs(): unmount(/data) succeeded
                        */
                        LINFO << __FUNCTION__ << "(): unmount(" << target << ") succeeded";
                        break;
                    }
                    PERROR << __FUNCTION__ << "(): umount(" << target << ") failed";
                    if (retry_count) sleep(1);
                }
                if (!umounted) { // umount失败，一般走不到这一步
                    // boot may fail but continue and leave it to later stage for now.
                    PERROR << __FUNCTION__ << "(): umount(" << target << ") timed out";
                    *fs_stat |= FS_STAT_RO_UNMOUNT_FAILED;
                }
            } else {
                *fs_stat |= FS_STAT_RO_MOUNT_FAILED;
            }
        }

        /*
        * Some system images do not have e2fsck for licensing reasons
        * (e.g. recent SDK system images). Detect these and skip the check.
        */
        if (access(E2FSCK_BIN, X_OK)) {
            LINFO << "Not running " << E2FSCK_BIN << " on " << blk_device
                << " (executable not in system image)";
        } else {
            /*
                log5：e2fsck -f -y 开始修复分区
                    init: [libfs_mgr]Running /system/bin/e2fsck on /dev/block/bootdevice/by-name/userdata
                    e2fsck: e2fsck 1.43.3 (04-Sep-2016)
                    e2fsck: Pass 1: Checking inodes, blocks, and sizes
                    e2fsck: Pass 2: Checking directory structure
                    e2fsck: Pass 3: Checking directory connectivity
                    e2fsck: Pass 4: Checking reference counts
                    e2fsck: Pass 5: Checking group summary information
                    e2fsck: /dev/block/bootdevice/by-name/userdata: 16299/1684256 files (5.2% non-contiguous), 540793/6731255 blocks
            */
            LINFO << "Running " << E2FSCK_BIN << " on " << blk_device;
            if (should_force_check(*fs_stat)) {
                ret = android_fork_execvp_ext(
                    ARRAY_SIZE(e2fsck_forced_argv), const_cast<char**>(e2fsck_forced_argv), &status,
                    true, LOG_KLOG | LOG_FILE, true, const_cast<char*>(FSCK_LOG_FILE), NULL, 0);
            } else {
                ret = android_fork_execvp_ext(
                    ARRAY_SIZE(e2fsck_argv), const_cast<char**>(e2fsck_argv), &status, true,
                    LOG_KLOG | LOG_FILE, true, const_cast<char*>(FSCK_LOG_FILE), NULL, 0);
            }

            if (ret < 0) {
                /* No need to check for error in fork, we can't really handle it now */
                LERROR << "Failed trying to run " << E2FSCK_BIN;
                *fs_stat |= FS_STAT_E2FSCK_FAILED;
            } else if (status != 0) {
                LINFO << "e2fsck returned status 0x" << std::hex << status;
                *fs_stat |= FS_STAT_E2FSCK_FS_FIXED;
            }
        }
    } else if (!strcmp(fs_type, "f2fs")) {
            const char *f2fs_fsck_argv[] = {
                    F2FS_FSCK_BIN,
                    "-a",
                    blk_device
            };
        LINFO << "Running " << F2FS_FSCK_BIN << " -a " << blk_device;

        ret = android_fork_execvp_ext(ARRAY_SIZE(f2fs_fsck_argv),
                                    const_cast<char **>(f2fs_fsck_argv),
                                    &status, true, LOG_KLOG | LOG_FILE,
                                    true, const_cast<char *>(FSCK_LOG_FILE),
                                    NULL, 0);
        if (ret < 0) {
            /* No need to check for error in fork, we can't really handle it now */
            LERROR << "Failed trying to run " << F2FS_FSCK_BIN;
        }
    }

    return;
}

/*
* __mount(): wrapper around the mount() system call which also
* sets the underlying block device to read-only if the mount is read-only.
* See "man 2 mount" for return values.
*/
static int __mount(const char *source, const char *target, const struct fstab_rec *rec)
{
    unsigned long mountflags = rec->flags;
    int ret;
    int save_errno;

    /* We need this because sometimes we have legacy symlinks
    * that are lingering around and need cleaning up.
    */
    struct stat info;
    if (!lstat(target, &info))
        if ((info.st_mode & S_IFMT) == S_IFLNK)
            unlink(target);
    mkdir(target, 0755);
    errno = 0;
    ret = mount(source, target, rec->fs_type, mountflags, rec->fs_options);
    save_errno = errno;
    /*
        log8：打印挂载成功
            init: [libfs_mgr]__mount(source=/dev/block/bootdevice/by-name/userdata,target=/data,type=ext4)=0: Success
    */
    PINFO << __FUNCTION__ << "(source=" << source << ",target=" << target
        << ",type=" << rec->fs_type << ")=" << ret;
    if ((ret == 0) && (mountflags & MS_RDONLY) != 0) {
        fs_mgr_set_blk_ro(source);
    }
    errno = save_errno;
    return ret;
}


```