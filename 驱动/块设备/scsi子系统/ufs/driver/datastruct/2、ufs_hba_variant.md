```c
 404 /**
 405  * struct ufs_hba_variant - variant specific parameters
 406  * @name: variant name
 407  */
 408 struct ufs_hba_variant {
 409     struct device               *dev;
 410     const char              *name;
 411     struct ufs_hba_variant_ops      *vops;
 412     struct ufs_hba_crypto_variant_ops   *crypto_vops;
 413     struct ufs_hba_pm_qos_variant_ops   *pm_qos_vops;
 414 };
```

