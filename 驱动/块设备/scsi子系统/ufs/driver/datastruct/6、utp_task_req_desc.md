```c
472 /**
473  * struct utp_task_req_desc - UTMRD structure
474  * @header: UTMRD header DW-0 to DW-3
475  * @task_req_upiu: Pointer to task request UPIU DW-4 to DW-11
476  * @task_rsp_upiu: Pointer to task response UPIU DW12 to DW-19
477  */
478 struct utp_task_req_desc {
479
480     /* DW 0-3 */
481     struct request_desc_header header;
482
483     /* DW 4-11 */
484     __le32 task_req_upiu[TASK_REQ_UPIU_SIZE_DWORDS];
485
486     /* DW 12-19 */
487     __le32 task_rsp_upiu[TASK_RSP_UPIU_SIZE_DWORDS];
488 };
```

