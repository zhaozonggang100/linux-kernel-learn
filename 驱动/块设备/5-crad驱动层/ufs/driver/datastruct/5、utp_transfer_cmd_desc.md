```c
402 /**
403  * struct ufshcd_sg_entry - UFSHCI PRD Entry
404  * @base_addr: Lower 32bit physical address DW-0
405  * @upper_addr: Upper 32bit physical address DW-1
406  * @reserved: Reserved for future use DW-2
407  * @size: size of physical segment DW-3
408  */
    /*
    	一个PRDT块的大小不能超过64kb
    */
409 struct ufshcd_sg_entry {
410     __le32    base_addr;
411     __le32    upper_addr;
412     __le32    reserved;
413     __le32    size;
414 };


416 /**
417  * struct utp_transfer_cmd_desc - UFS Command Descriptor structure
418  * @command_upiu: Command UPIU Frame address
419  * @response_upiu: Response UPIU Frame address
420  * @prd_table: Physical Region Descriptor
421  */
422 struct utp_transfer_cmd_desc {
423     u8 command_upiu[ALIGNED_UPIU_SIZE];
424     u8 response_upiu[ALIGNED_UPIU_SIZE];
425     struct ufshcd_sg_entry    prd_table[SG_ALL];	// SG_ALL：128，一个UTRD中PRDT数组的长度
426 };
```

