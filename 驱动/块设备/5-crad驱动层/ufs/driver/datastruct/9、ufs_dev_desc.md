```c
534 /**
535  * ufs_dev_desc - ufs device details from the device descriptor
536  *
537  * @wmanufacturerid: card details
538  * @model: card model
539  */
540 struct ufs_dev_desc {
541     u16 wmanufacturerid;
542     char model[MAX_MODEL_LEN + 1];
543     u16 wspecversion;
544 };
```

