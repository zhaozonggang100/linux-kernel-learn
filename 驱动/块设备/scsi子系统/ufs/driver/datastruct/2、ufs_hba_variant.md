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

 305 /**
 306  * struct ufs_hba_variant_ops - variant specific callbacks
 307  * @init: called when the driver is initialized
 308  * @exit: called to cleanup everything done in init
 309  * @get_ufs_hci_version: called to get UFS HCI version
 310  * @clk_scale_notify: notifies that clks are scaled up/down
 311  * @setup_clocks: called before touching any of the controller registers
 312  * @setup_regulators: called before accessing the host controller
 313  * @hce_enable_notify: called before and after HCE enable bit is set to allow
 314  *                     variant specific Uni-Pro initialization.
 315  * @link_startup_notify: called before and after Link startup is carried out
 316  *                       to allow variant specific Uni-Pro initialization.
 317  * @pwr_change_notify: called before and after a power mode change
 318  *          is carried out to allow vendor spesific capabilities
 319  *          to be set.
 320  * @setup_xfer_req: called before any transfer request is issued
 321  *                  to set some things
 322  * @setup_task_mgmt: called before any task management request is issued
 323  *                  to set some things
 324  * @hibern8_notify: called around hibern8 enter/exit
 325  * @apply_dev_quirks: called to apply device specific quirks
 326  * @suspend: called during host controller PM callback
 327  * @resume: called during host controller PM callback
 328  * @full_reset:  called during link recovery for handling variant specific
 329  *       implementations of resetting the hci
 330  * @update_sec_cfg: called to restore host controller secure configuration
 331  * @get_scale_down_gear: called to get the minimum supported gear to
 332  *           scale down
 333  * @set_bus_vote: called to vote for the required bus bandwidth
 334  * @phy_initialization: used to initialize phys
 335  */
 336 struct ufs_hba_variant_ops {
 337     int (*init)(struct ufs_hba *);
 338     void    (*exit)(struct ufs_hba *);
 339     u32 (*get_ufs_hci_version)(struct ufs_hba *);
 340     int (*clk_scale_notify)(struct ufs_hba *, bool,
 341                     enum ufs_notify_change_status);
 342     int (*setup_clocks)(struct ufs_hba *, bool,
 343                 enum ufs_notify_change_status);
 344     int     (*setup_regulators)(struct ufs_hba *, bool);
 345     int (*hce_enable_notify)(struct ufs_hba *,
 346                      enum ufs_notify_change_status);
 347     int (*link_startup_notify)(struct ufs_hba *,
 348                        enum ufs_notify_change_status);
 349     int (*pwr_change_notify)(struct ufs_hba *,
 350                     enum ufs_notify_change_status status,
 351                     struct ufs_pa_layer_attr *,
 352                     struct ufs_pa_layer_attr *);
 353     void    (*setup_xfer_req)(struct ufs_hba *, int, bool);
 354     void    (*setup_task_mgmt)(struct ufs_hba *, int, u8);
 355     void    (*hibern8_notify)(struct ufs_hba *, enum uic_cmd_dme,
 356                     enum ufs_notify_change_status);
  357     int (*apply_dev_quirks)(struct ufs_hba *);
 358     int     (*suspend)(struct ufs_hba *, enum ufs_pm_op);
 359     int     (*resume)(struct ufs_hba *, enum ufs_pm_op);
 360     int (*full_reset)(struct ufs_hba *);
 361     void    (*dbg_register_dump)(struct ufs_hba *hba, bool no_sleep);
 362     int (*update_sec_cfg)(struct ufs_hba *hba, bool restore_sec_cfg);
 363     u32 (*get_scale_down_gear)(struct ufs_hba *);
 364     int (*set_bus_vote)(struct ufs_hba *, bool);
 365     int (*phy_initialization)(struct ufs_hba *);
 366 #ifdef CONFIG_DEBUG_FS
 367     void    (*add_debugfs)(struct ufs_hba *hba, struct dentry *root);
 368     void    (*remove_debugfs)(struct ufs_hba *hba);
 369 #endif
 370 };
```

