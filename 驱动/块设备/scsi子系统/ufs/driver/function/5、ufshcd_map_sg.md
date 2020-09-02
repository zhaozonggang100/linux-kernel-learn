### 1、file

```
drivers/scsi/ufs/ufshcd.c
```

### 2、定义

```c
 3270 /**
 3271  * ufshcd_map_sg - Map scatter-gather list to prdt
 3272  * @lrbp - pointer to local reference block
 3273  *
 3274  * Returns 0 in case of success, non-zero value in case of failure
 3275  */
 3276 static int ufshcd_map_sg(struct ufs_hba *hba, struct ufshcd_lrb *lrbp)
 3277 {
 3278     struct ufshcd_sg_entry *prd_table;
 3279     struct scatterlist *sg;
 3280     struct scsi_cmnd *cmd;
 3281     int sg_segments;
 3282     int i;
 3283
 3284     cmd = lrbp->cmd;
 3285     sg_segments = scsi_dma_map(cmd);
 3286     if (sg_segments < 0)
 3287         return sg_segments;
 3288
 3289     if (sg_segments) {
 3290         if (hba->quirks & UFSHCD_QUIRK_PRDT_BYTE_GRAN)
 3291             lrbp->utr_descriptor_ptr->prd_table_length =
 3292                 cpu_to_le16((u16)(sg_segments *
 3293                     sizeof(struct ufshcd_sg_entry)));
 3294         else
 3295             lrbp->utr_descriptor_ptr->prd_table_length =
 3296                 cpu_to_le16((u16) (sg_segments));
 3297
 3298         prd_table = (struct ufshcd_sg_entry *)lrbp->ucd_prdt_ptr;
 3299
 3300         scsi_for_each_sg(cmd, sg, sg_segments, i) {
 3301             prd_table[i].size  =
 3302                 cpu_to_le32(((u32) sg_dma_len(sg))-1);
 3303             prd_table[i].base_addr =
 3304                 cpu_to_le32(lower_32_bits(sg->dma_address));
 3305             prd_table[i].upper_addr =
 3306                 cpu_to_le32(upper_32_bits(sg->dma_address));
 3307             prd_table[i].reserved = 0;
 3308         }
 3309     } else {
 3310         lrbp->utr_descriptor_ptr->prd_table_length = 0;
 3311     }
 3312
 3313     return 0;
 3314 }
```

