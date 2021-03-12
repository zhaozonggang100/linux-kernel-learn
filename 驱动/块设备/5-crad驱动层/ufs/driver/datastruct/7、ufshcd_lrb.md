```c
 180 /**
 181  * struct ufshcd_lrb - local reference block
 182  * @utr_descriptor_ptr: UTRD address of the command
 183  * @ucd_req_ptr: UCD address of the command
 184  * @ucd_rsp_ptr: Response UPIU address for this command
 185  * @ucd_prdt_ptr: PRDT address of the command
 186  * @utrd_dma_addr: UTRD dma address for debug
 187  * @ucd_prdt_dma_addr: PRDT dma address for debug
 188  * @ucd_rsp_dma_addr: UPIU response dma address for debug
 189  * @ucd_req_dma_addr: UPIU request dma address for debug
 190  * @cmd: pointer to SCSI command
 191  * @sense_buffer: pointer to sense buffer address of the SCSI command
 192  * @sense_bufflen: Length of the sense buffer
 193  * @scsi_status: SCSI status of the command
 194  * @command_type: SCSI, UFS, Query.
 195  * @task_tag: Task tag of the command
 196  * @lun: LUN of the command
 197  * @intr_cmd: Interrupt command (doesn't participate in interrupt aggregation)
 198  * @issue_time_stamp: time stamp for debug purposes
 199  * @complete_time_stamp: time stamp for statistics
 200  * @req_abort_skip: skip request abort task flag
 201  */
 202 struct ufshcd_lrb {
 203     struct utp_transfer_req_desc *utr_descriptor_ptr;
 204     struct utp_upiu_req *ucd_req_ptr;
 205     struct utp_upiu_rsp *ucd_rsp_ptr;
 206     struct ufshcd_sg_entry *ucd_prdt_ptr;
 207
 208     dma_addr_t utrd_dma_addr;
 209     dma_addr_t ucd_req_dma_addr;
 210     dma_addr_t ucd_rsp_dma_addr;
 211     dma_addr_t ucd_prdt_dma_addr;
 212
 213     struct scsi_cmnd *cmd;
 214     u8 *sense_buffer;
 215     unsigned int sense_bufflen;
 216     int scsi_status;
 217
 218     int command_type;
 219     int task_tag;
 220     u8 lun; /* UPIU LUN id field is only 8-bit wide */
 221     bool intr_cmd;
 222     ktime_t issue_time_stamp;
 223     ktime_t complete_time_stamp;
 224
 225     bool req_abort_skip;
 226 };
```

