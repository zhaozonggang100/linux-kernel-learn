### 1、file

```
drivers/scsi/ufs/ufshcd.c
```

### 2、申明

在ufshcd.c中静态申明并定义

### 3、定义

```c
 3566 /**
 3567  * ufshcd_comp_scsi_upiu - UFS Protocol Information Unit(UPIU)
 3568  *             for SCSI Purposes
 3569  * @hba - per adapter instance
 3570  * @lrb - pointer to local reference block
 3571  */
 3572 static int ufshcd_comp_scsi_upiu(struct ufs_hba *hba, struct ufshcd_lrb *lrbp)
 3573 {
 3574     u32 upiu_flags;
 3575     int ret = 0;
 3576
 3577     if ((hba->ufs_version == UFSHCI_VERSION_10) ||
 3578         (hba->ufs_version == UFSHCI_VERSION_11))
 3579         lrbp->command_type = UTP_CMD_TYPE_SCSI;
 3580     else
 3581         lrbp->command_type = UTP_CMD_TYPE_UFS_STORAGE;
 3582
 3583     if (likely(lrbp->cmd)) {
 3584         ret = ufshcd_prepare_req_desc_hdr(hba, lrbp,
 3585                 &upiu_flags, lrbp->cmd->sc_data_direction);
 3586         ufshcd_prepare_utp_scsi_cmd_upiu(lrbp, upiu_flags);
 3587     } else {
 3588         ret = -EINVAL;
 3589     }
 3590
 3591     return ret;
 3592 }
```

