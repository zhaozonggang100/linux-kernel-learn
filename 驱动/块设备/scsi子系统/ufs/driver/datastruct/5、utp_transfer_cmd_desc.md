```c
416 /**
417  * struct utp_transfer_cmd_desc - UFS Command Descriptor structure
418  * @command_upiu: Command UPIU Frame address
419  * @response_upiu: Response UPIU Frame address
420  * @prd_table: Physical Region Descriptor
421  */
422 struct utp_transfer_cmd_desc {
423     u8 command_upiu[ALIGNED_UPIU_SIZE];
424     u8 response_upiu[ALIGNED_UPIU_SIZE];
425     struct ufshcd_sg_entry    prd_table[SG_ALL];
426 };
```

