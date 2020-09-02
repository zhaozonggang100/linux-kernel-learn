```c
 715 /**
 716  * struct ufs_hba - per adapter private structure
 717  * @mmio_base: UFSHCI base register address
 718  * @ucdl_base_addr: UFS Command Descriptor base address
 719  * @utrdl_base_addr: UTP Transfer Request Descriptor base address
 720  * @utmrdl_base_addr: UTP Task Management Descriptor base address
 721  * @ucdl_dma_addr: UFS Command Descriptor DMA address
 722  * @utrdl_dma_addr: UTRDL DMA address
 723  * @utmrdl_dma_addr: UTMRDL DMA address
 724  * @host: Scsi_Host instance of the driver
 725  * @dev: device handle
 726  * @lrb: local reference block
 727  * @lrb_in_use: lrb in use
 728  * @outstanding_tasks: Bits representing outstanding task requests
 729  * @outstanding_reqs: Bits representing outstanding transfer requests
 730  * @capabilities: UFS Controller Capabilities
 731  * @nutrs: Transfer Request Queue depth supported by controller
 732  * @nutmrs: Task Management Queue depth supported by controller
 733  * @ufs_version: UFS Version to which controller complies
 734  * @var: pointer to variant specific data
 735  * @priv: pointer to variant specific private data
 736  * @irq: Irq number of the controller
 737  * @active_uic_cmd: handle of active UIC command
 738  * @uic_cmd_mutex: mutex for uic command
 739  * @tm_wq: wait queue for task management
 740  * @tm_tag_wq: wait queue for free task management slots
 741  * @tm_slots_in_use: bit map of task management request slots in use
 742  * @pwr_done: completion for power mode change
 743  * @tm_condition: condition variable for task management
 744  * @ufshcd_state: UFSHCD states
 745  * @eh_flags: Error handling flags
 746  * @intr_mask: Interrupt Mask Bits
 747  * @ee_ctrl_mask: Exception event control mask
 748  * @is_powered: flag to check if HBA is powered
 749  * @eh_work: Worker to handle UFS errors that require s/w attention
 750  * @eeh_work: Worker to handle exception events
 751  * @errors: HBA errors
 752  * @uic_error: UFS interconnect layer error status
 753  * @saved_err: sticky error mask
 754  * @saved_uic_err: sticky UIC error mask
 755  * @dev_cmd: ufs device management command information
 756  * @last_dme_cmd_tstamp: time stamp of the last completed DME command
 757  * @auto_bkops_enabled: to track whether bkops is enabled in device
 758  * @ufs_stats: ufshcd statistics to be used via debugfs
 759  * @debugfs_files: debugfs files associated with the ufs stats
 760  * @ufshcd_dbg_print: Bitmask for enabling debug prints
 761  * @extcon: pointer to external connector device
 762  * @card_detect_nb: card detector notifier registered with @extcon
 763  * @card_detect_work: work to exectute the card detect function
 764  * @card_state: card state event, enum ufshcd_card_state defines possible states
 765  * @vreg_info: UFS device voltage regulator information
 766  * @clk_list_head: UFS host controller clocks list node head
 767  * @pwr_info: holds current power mode
 768  * @max_pwr_info: keeps the device max valid pwm
 769  * @hibern8_on_idle: UFS Hibern8 on idle related data
 770  * @desc_size: descriptor sizes reported by device
 771  * @urgent_bkops_lvl: keeps track of urgent bkops level for device
 772  * @is_urgent_bkops_lvl_checked: keeps track if the urgent bkops level for
 773  *  device is known or not.
 774  * @scsi_block_reqs_cnt: reference counting for scsi block requests
 775  */
  776 struct ufs_hba {
 777     void __iomem *mmio_base;
 778
 779     /* Virtual memory reference */
 780     struct utp_transfer_cmd_desc *ucdl_base_addr;
 781     struct utp_transfer_req_desc *utrdl_base_addr;
 782     struct utp_task_req_desc *utmrdl_base_addr;
 783
 784     /* DMA memory reference */
 785     dma_addr_t ucdl_dma_addr;
 786     dma_addr_t utrdl_dma_addr;
 787     dma_addr_t utmrdl_dma_addr;
 788
 789     struct Scsi_Host *host;
 790     struct device *dev;
 791     /*
 792      * This field is to keep a reference to "scsi_device" corresponding to
 793      * "UFS device" W-LU.
 794      */
 795     struct scsi_device *sdev_ufs_device;
 796
 797     enum ufs_dev_pwr_mode curr_dev_pwr_mode;
 798     enum uic_link_state uic_link_state;
 799     /* Desired UFS power management level during runtime PM */
 800     int rpm_lvl;
 801     /* Desired UFS power management level during system PM */
 802     int spm_lvl;
 803     struct device_attribute rpm_lvl_attr;
 804     struct device_attribute spm_lvl_attr;
 805     int pm_op_in_progress;
 806
 807     struct ufshcd_lrb *lrb;
 808     unsigned long lrb_in_use;
 809
 810     unsigned long outstanding_tasks;
 811     unsigned long outstanding_reqs;
 812
 813     u32 capabilities;
 814     int nutrs;
 815     int nutmrs;
 816     u32 ufs_version;
 817     struct ufs_hba_variant *var;
 818     void *priv;
 819     unsigned int irq;
 820     bool is_irq_enabled;
 821     bool crash_on_err;
 822
 823     u32 dev_ref_clk_gating_wait;
 824     u32 dev_ref_clk_freq;
 825
 826     /* Interrupt aggregation support is broken */
 827     #define UFSHCD_QUIRK_BROKEN_INTR_AGGR           UFS_BIT(0)
 828
 829     /*
 830      * delay before each dme command is required as the unipro
 831      * layer has shown instabilities
 832      */
 833     #define UFSHCD_QUIRK_DELAY_BEFORE_DME_CMDS      UFS_BIT(1)
 834
 835     /*
 836      * If UFS host controller is having issue in processing LCC (Line
 837      * Control Command) coming from device then enable this quirk.
 838      * When this quirk is enabled, host controller driver should disable
 839      * the LCC transmission on UFS device (by clearing TX_LCC_ENABLE
  840      * attribute of device to 0).
 841      */
 842     #define UFSHCD_QUIRK_BROKEN_LCC             UFS_BIT(2)
 843
 844     /*
 845      * The attribute PA_RXHSUNTERMCAP specifies whether or not the
 846      * inbound Link supports unterminated line in HS mode. Setting this
 847      * attribute to 1 fixes moving to HS gear.
 848      */
 849     #define UFSHCD_QUIRK_BROKEN_PA_RXHSUNTERMCAP        UFS_BIT(3)
 850
 851     /*
 852      * This quirk needs to be enabled if the host contoller only allows
 853      * accessing the peer dme attributes in AUTO mode (FAST AUTO or
 854      * SLOW AUTO).
 855      */
 856     #define UFSHCD_QUIRK_DME_PEER_ACCESS_AUTO_MODE      UFS_BIT(4)
 857
 858     /*
 859      * This quirk needs to be enabled if the host contoller doesn't
 860      * advertise the correct version in UFS_VER register. If this quirk
 861      * is enabled, standard UFS host driver will call the vendor specific
 862      * ops (get_ufs_hci_version) to get the correct version.
 863      */
 864     #define UFSHCD_QUIRK_BROKEN_UFS_HCI_VERSION     UFS_BIT(5)
 865
 866     /*
 867      * This quirk needs to be enabled if the host contoller regards
 868      * resolution of the values of PRDTO and PRDTL in UTRD as byte.
 869      */
 870     #define UFSHCD_QUIRK_PRDT_BYTE_GRAN         UFS_BIT(7)
 871
 872     /* HIBERN8 support is broken */
 873     #define UFSHCD_QUIRK_BROKEN_HIBERN8         UFS_BIT(8)
 874
 875     /*
 876      * UFS controller version register (VER) wrongly advertise the version
 877      * as v1.0 though controller implementation is as per UFSHCI v1.1
 878      * specification.
 879      */
 880     #define UFSHCD_QUIRK_BROKEN_VER_REG_1_1         UFS_BIT(9)
 881
 882     /* UFSHC advertises 64-bit not supported even though it supports */
 883     #define UFSHCD_QUIRK_BROKEN_CAP_64_BIT_0        UFS_BIT(10)
 884
 885     /*
 886      * If LCC (Line Control Command) are having issue on the host
 887      * controller then enable this quirk. Note that connected UFS device
 888      * should also have workaround to not expect LCC commands from host.
 889      */
 890     #define UFSHCD_BROKEN_LCC               UFS_BIT(11)
 891
 892     /*
 893      * If UFS device is having issue in processing LCC (Line Control
 894      * Command) coming from UFS host controller then enable this quirk.
 895      * When this quirk is enabled, host controller driver should disable
 896      * the LCC transmission on UFS host controller (by clearing
 897      * TX_LCC_ENABLE attribute of host to 0).
 898      */
 899     #define UFSHCD_BROKEN_LCC_PROCESSING_ON_DEVICE      UFS_BIT(12)
  900
 901     #define UFSHCD_BROKEN_LCC_PROCESSING_ON_HOST        UFS_BIT(13)
 902
 903     #define UFSHCD_QUIRK_DME_PEER_GET_FAST_MODE     UFS_BIT(14)
 904
 905     /* Auto hibern8 support is broken */
 906     #define UFSHCD_QUIRK_BROKEN_AUTO_HIBERN8        UFS_BIT(15)
 907
 908     unsigned int quirks;    /* Deviations from standard UFSHCI spec. */
 909
 910     wait_queue_head_t tm_wq;
 911     wait_queue_head_t tm_tag_wq;
 912     unsigned long tm_condition;
 913     unsigned long tm_slots_in_use;
 914
 915     struct uic_command *active_uic_cmd;
 916     struct mutex uic_cmd_mutex;
 917     struct completion *uic_async_done;
 918
 919     u32 ufshcd_state;
 920     u32 eh_flags;
 921     u32 intr_mask;
 922     u16 ee_ctrl_mask;
 923     bool is_powered;
 924
 925     /* Work Queues */
 926     struct work_struct eh_work;
 927     struct work_struct eeh_work;
 928     struct work_struct rls_work;
 929
 930     /* HBA Errors */
 931     u32 errors;
 932     u32 uic_error;
 933     u32 ce_error;   /* crypto engine errors */
 934     u32 saved_err;
 935     u32 saved_uic_err;
 936     u32 saved_ce_err;
 937     bool silence_err_logs;
 938     bool force_host_reset;
 939     bool auto_h8_err;
 940     struct ufs_stats ufs_stats;
 941
 942     /* Device management request data */
 943     struct ufs_dev_cmd dev_cmd;
 944     ktime_t last_dme_cmd_tstamp;
 945
 946     /* Keeps information of the UFS device connected to this host */
 947     struct ufs_dev_info dev_info;
 948     bool auto_bkops_enabled;
 949
 950 #ifdef CONFIG_DEBUG_FS
 951     struct debugfs_files debugfs_files;
 952 #endif
 953
 954     struct ufs_vreg_info vreg_info;
 955     struct list_head clk_list_head;
  956
 957     bool wlun_dev_clr_ua;
 958
 959     /* Number of requests aborts */
 960     int req_abort_count;
 961
 962     /* Number of lanes available (1 or 2) for Rx/Tx */
 963     u32 lanes_per_direction;
 964
 965     /* Gear limits */
 966     u32 limit_tx_hs_gear;
 967     u32 limit_rx_hs_gear;
 968     u32 limit_tx_pwm_gear;
 969     u32 limit_rx_pwm_gear;
 970
 971     u32 scsi_cmd_timeout;
 972
 973     /* Bitmask for enabling debug prints */
 974     u32 ufshcd_dbg_print;
 975
 976     struct extcon_dev *extcon;
 977     struct notifier_block card_detect_nb;
 978     struct work_struct card_detect_work;
 979     atomic_t card_state;
 980
 981     struct ufs_pa_layer_attr pwr_info;
 982     struct ufs_pwr_mode_info max_pwr_info;
 983
 984     struct ufs_clk_gating clk_gating;
 985     struct ufs_hibern8_on_idle hibern8_on_idle;
 986     struct ufshcd_cmd_log cmd_log;
 987
 988     /* Control to enable/disable host capabilities */
 989     u32 caps;
 990     /* Allow dynamic clk gating */
 991 #define UFSHCD_CAP_CLK_GATING   (1 << 0)
 992     /* Allow hiberb8 with clk gating */
 993 #define UFSHCD_CAP_HIBERN8_WITH_CLK_GATING (1 << 1)
 994     /* Allow dynamic clk scaling */
 995 #define UFSHCD_CAP_CLK_SCALING  (1 << 2)
 996     /* Allow auto bkops to enabled during runtime suspend */
 997 #define UFSHCD_CAP_AUTO_BKOPS_SUSPEND (1 << 3)
 998     /*
 999      * This capability allows host controller driver to use the UFS HCI's
1000      * interrupt aggregation capability.
1001      * CAUTION: Enabling this might reduce overall UFS throughput.
1002      */
1003 #define UFSHCD_CAP_INTR_AGGR (1 << 4)
1004     /* Allow standalone Hibern8 enter on idle */
1005 #define UFSHCD_CAP_HIBERN8_ENTER_ON_IDLE (1 << 5)
1006
1007     /*
1008      * This capability allows the device auto-bkops to be always enabled
1009      * except during suspend (both runtime and suspend).
1010      * Enabling this capability means that device will always be allowed
1011      * to do background operation when it's active but it might degrade
1012      * the performance of ongoing read/write operations.
1013      */
1014 #define UFSHCD_CAP_KEEP_AUTO_BKOPS_ENABLED_EXCEPT_SUSPEND (1 << 6)
1015     /*
1016      * If host controller hardware can be power collapsed when UFS link is
1017      * in hibern8 then enable this cap.
1018      */
1019 #define UFSHCD_CAP_POWER_COLLAPSE_DURING_HIBERN8 (1 << 7)
1020
1021     struct devfreq *devfreq;
1022     struct ufs_clk_scaling clk_scaling;
1023     bool is_sys_suspended;
1024
1025     enum bkops_status urgent_bkops_lvl;
1026     bool is_urgent_bkops_lvl_checked;
1027
1028     /* sync b/w diff contexts */
1029     struct rw_semaphore lock;
1030     unsigned long shutdown_in_prog;
1031
1032     /* If set, don't gate device ref_clk during clock gating */
1033     bool no_ref_clk_gating;
1034
1035     int scsi_block_reqs_cnt;
1036
1037     bool full_init_linereset;
1038     struct pinctrl *pctrl;
1039
1040     struct reset_control *core_reset;
1041
1042     struct ufs_desc_size desc_size;
1043     bool restore_needed;
1044
1045     int latency_hist_enabled;
1046     struct io_latency_state io_lat_s;
1047
1048     bool reinit_g4_rate_A;
1049     bool force_g4;
1050     /* distinguish between resume and restore */
1051     bool restore;
1052 };
```

