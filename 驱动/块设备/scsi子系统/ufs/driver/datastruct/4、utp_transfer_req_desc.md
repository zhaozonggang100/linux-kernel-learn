

```c
430 /**
431  * struct request_desc_header - Descriptor Header common to both UTRD and UTMRD
432  * @dword0: Descriptor Header DW0
433  * @dword1: Descriptor Header DW1
434  * @dword2: Descriptor Header DW2
435  * @dword3: Descriptor Header DW3
436  */
437 struct request_desc_header {
438     __le32 dword_0;
439     __le32 dword_1;
440     __le32 dword_2;
441     __le32 dword_3;
442 };

444 /**
445  * struct utp_transfer_req_desc - UTRD structure
446  * @header: UTRD header DW-0 to DW-3
447  * @command_desc_base_addr_lo: UCD base address low DW-4
448  * @command_desc_base_addr_hi: UCD base address high DW-5
449  * @response_upiu_length: response UPIU length DW-6
450  * @response_upiu_offset: response UPIU offset DW-6
451  * @prd_table_length: Physical region descriptor length DW-7
452  * @prd_table_offset: Physical region descriptor offset DW-7
453  */
454 struct utp_transfer_req_desc {
455
456     /* DW 0-3 */
457     struct request_desc_header header;
458
459     /* DW 4-5*/
460     __le32  command_desc_base_addr_lo;
461     __le32  command_desc_base_addr_hi;
462
463     /* DW 6 */
464     __le16  response_upiu_length;
465     __le16  response_upiu_offset;
466
467     /* DW 7 */
468     __le16  prd_table_length;
469     __le16  prd_table_offset;
470 };
```

