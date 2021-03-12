### 1、file

```
drivers/scsi/ufs/ufshcd.c
```

### 2、申明

由于ufs向上需要注册给scsi子系统（u盘也是），所以需要实现scsi子系统的模板结构scsi_host_template

```c
 9390 static struct scsi_host_template ufshcd_driver_template = {
 9391     .module         = THIS_MODULE,
 9392     .name           = UFSHCD,
 9393     .proc_name      = UFSHCD,
 9394     .queuecommand       = ufshcd_queuecommand,
 9395     .slave_alloc        = ufshcd_slave_alloc,
 9396     .slave_configure    = ufshcd_slave_configure,
 9397     .slave_destroy      = ufshcd_slave_destroy,
 9398     .change_queue_depth = ufshcd_change_queue_depth,
 9399     .eh_abort_handler   = ufshcd_abort,
 9400     .eh_device_reset_handler = ufshcd_eh_device_reset_handler,
 9401     .eh_host_reset_handler   = ufshcd_eh_host_reset_handler,
 9402     .eh_timed_out       = ufshcd_eh_timed_out,
 9403     .ioctl          = ufshcd_ioctl,
 9404 #ifdef CONFIG_COMPAT
 9405     .compat_ioctl       = ufshcd_ioctl,
 9406 #endif
 9407     .this_id        = -1,
 9408     .sg_tablesize       = SG_ALL,
 9409     .cmd_per_lun        = UFSHCD_CMD_PER_LUN,
 9410     .can_queue      = UFSHCD_CAN_QUEUE,
 9411     .max_host_blocked   = 1,
 9412     .track_queue_depth  = 1,
 9413 };
```

### 3、定义

```c
 3675 /**
 3676  * ufshcd_queuecommand - main entry point for SCSI requests
 3677  * @cmd: command from SCSI Midlayer
 3678  * @done: call back function
 3679  *
 3680  * Returns 0 for success, non-zero in case of failure
 3681  */
 3682 static int ufshcd_queuecommand(struct Scsi_Host *host, struct scsi_cmnd *cmd)
 3683 {
 3684     struct ufshcd_lrb *lrbp;
 3685     struct ufs_hba *hba;
 3686     unsigned long flags;
 3687     int tag;
 3688     int err = 0;
 3689     bool has_read_lock = false;
 3690
 3691     hba = shost_priv(host);
 3692
 3693     if (!cmd || !cmd->request || !hba)
 3694         return -EINVAL;
 3695
 3696     tag = cmd->request->tag;
 3697     if (!ufshcd_valid_tag(hba, tag)) {
 3698         dev_err(hba->dev,
 3699             "%s: invalid command tag %d: cmd=0x%p, cmd->request=0x%p",
 3700             __func__, tag, cmd, cmd->request);
 3701         BUG();
 3702     }
 3703
 3704     err = ufshcd_get_read_lock(hba, cmd->device->lun);
 3705     if (unlikely(err < 0)) {
 3706         if (err == -EPERM) {
 3707             if (!ufshcd_vops_crypto_engine_get_req_status(hba)) {
 3708                 set_host_byte(cmd, DID_ERROR);
 3709                 cmd->scsi_done(cmd);
 3710                 return 0;
 3711             } else {
 3712                 return SCSI_MLQUEUE_HOST_BUSY;
 3713             }
 3714         }
 3715         if (err == -EAGAIN)
 3716             return SCSI_MLQUEUE_HOST_BUSY;
 3717     } else if (err == 1) {
 3718         has_read_lock = true;
 3719     }
 3720
 3721     /*
 3722      * err might be non-zero here but logic later in this function
 3723      * assumes that err is set to 0.
 3724      */
 3725     err = 0;
 3726
 3727     spin_lock_irqsave(hba->host->host_lock, flags);
  3728
 3729     /* if error handling is in progress, return host busy */
 3730     if (ufshcd_eh_in_progress(hba)) {
 3731         err = SCSI_MLQUEUE_HOST_BUSY;
 3732         goto out_unlock;
 3733     }
 3734
 3735     if (hba->extcon && ufshcd_is_card_offline(hba)) {
 3736         set_host_byte(cmd, DID_BAD_TARGET);
 3737         cmd->scsi_done(cmd);
 3738         goto out_unlock;
 3739     }
 3740
 3741     switch (hba->ufshcd_state) {
 3742     case UFSHCD_STATE_OPERATIONAL:
 3743         break;
 3744     case UFSHCD_STATE_EH_SCHEDULED:
 3745     case UFSHCD_STATE_RESET:
 3746         err = SCSI_MLQUEUE_HOST_BUSY;
 3747         goto out_unlock;
 3748     case UFSHCD_STATE_ERROR:
 3749         set_host_byte(cmd, DID_ERROR);
 3750         cmd->scsi_done(cmd);
 3751         goto out_unlock;
 3752     default:
 3753         dev_WARN_ONCE(hba->dev, 1, "%s: invalid state %d\n",
 3754                 __func__, hba->ufshcd_state);
 3755         set_host_byte(cmd, DID_BAD_TARGET);
 3756         cmd->scsi_done(cmd);
 3757         goto out_unlock;
 3758     }
 3759     spin_unlock_irqrestore(hba->host->host_lock, flags);
 3760
 3761     hba->req_abort_count = 0;
 3762
 3763     /* acquire the tag to make sure device cmds don't use it */
 3764     if (test_and_set_bit_lock(tag, &hba->lrb_in_use)) {
 3765         /*
 3766          * Dev manage command in progress, requeue the command.
 3767          * Requeuing the command helps in cases where the request *may*
 3768          * find different tag instead of waiting for dev manage command
 3769          * completion.
 3770          */
 3771         err = SCSI_MLQUEUE_HOST_BUSY;
 3772         goto out;
 3773     }
 3774
 3775     hba->ufs_stats.clk_hold.ctx = QUEUE_CMD;
 3776     err = ufshcd_hold(hba, true);
 3777     if (err) {
 3778         err = SCSI_MLQUEUE_HOST_BUSY;
 3779         clear_bit_unlock(tag, &hba->lrb_in_use);
 3780         goto out;
 3781     }
 3782     if (ufshcd_is_clkgating_allowed(hba))
 3783         WARN_ON(hba->clk_gating.state != CLKS_ON);
 3784
 3785     err = ufshcd_hibern8_hold(hba, true);
 3786     if (err) {
 3787         clear_bit_unlock(tag, &hba->lrb_in_use);
  3788         err = SCSI_MLQUEUE_HOST_BUSY;
 3789         hba->ufs_stats.clk_rel.ctx = QUEUE_CMD;
 3790         ufshcd_release(hba, true);
 3791         goto out;
 3792     }
 3793
 3794     if (ufshcd_is_hibern8_on_idle_allowed(hba))
 3795         WARN_ON(hba->hibern8_on_idle.state != HIBERN8_EXITED);
 3796
 3797     /* Vote PM QoS for the request */
 3798     ufshcd_vops_pm_qos_req_start(hba, cmd->request);
 3799
 3800     /* IO svc time latency histogram */
 3801     if (hba != NULL && cmd->request != NULL) {
 3802         if (hba->latency_hist_enabled) {
 3803             switch (req_op(cmd->request)) {
 3804             case REQ_OP_READ:
 3805             case REQ_OP_WRITE:
 3806                 cmd->request->lat_hist_io_start = ktime_get();
 3807                 cmd->request->lat_hist_enabled = 1;
 3808             }
 3809         } else
 3810             cmd->request->lat_hist_enabled = 0;
 3811     }
 3812
 3813     WARN_ON(hba->clk_gating.state != CLKS_ON);
 3814
 3815     lrbp = &hba->lrb[tag];
 3816
 3817     WARN_ON(lrbp->cmd);
 3818     lrbp->cmd = cmd;
 3819     lrbp->sense_bufflen = UFSHCD_REQ_SENSE_SIZE;
 3820     lrbp->sense_buffer = cmd->sense_buffer;
 3821     lrbp->task_tag = tag;
 3822     lrbp->lun = ufshcd_scsi_to_upiu_lun(cmd->device->lun);
 3823     lrbp->intr_cmd = !ufshcd_is_intr_aggr_allowed(hba) ? true : false;
 3824     lrbp->req_abort_skip = false;
 3825
     	  /*
     	  		ufs传输层向uic层传送command
     	  */
 3826     err = ufshcd_comp_scsi_upiu(hba, lrbp);
 3827     if (err) {
 3828         if (err != -EAGAIN)
 3829             dev_err(hba->dev,
 3830                 "%s: failed to compose upiu %d cmd:0x%08x lun:%d\n",
 3831                 __func__, err, cmd, lrbp->lun);
 3832
 3833         lrbp->cmd = NULL;
 3834         clear_bit_unlock(tag, &hba->lrb_in_use);
 3835         ufshcd_release_all(hba);
 3836         ufshcd_vops_pm_qos_req_end(hba, cmd->request, true);
 3837         goto out;
 3838     }
 3839
     	  /*
     	  		sg map
     	  */
 3840     err = ufshcd_map_sg(hba, lrbp);
 3841     if (err) {
 3842         lrbp->cmd = NULL;
 3843         clear_bit_unlock(tag, &hba->lrb_in_use);
 3844         ufshcd_release_all(hba);
 3845         ufshcd_vops_pm_qos_req_end(hba, cmd->request, true);
 3846         goto out;
 3847     }
 3848
 3849     err = ufshcd_vops_crypto_engine_cfg_start(hba, tag);
 3850     if (err) {
  3851         if (err != -EAGAIN)
 3852             dev_err(hba->dev,
 3853                 "%s: failed to configure crypto engine %d\n",
 3854                 __func__, err);
 3855
 3856         scsi_dma_unmap(lrbp->cmd);
 3857         lrbp->cmd = NULL;
 3858         clear_bit_unlock(tag, &hba->lrb_in_use);
 3859         ufshcd_release_all(hba);
 3860         ufshcd_vops_pm_qos_req_end(hba, cmd->request, true);
 3861
 3862         goto out;
 3863     }
 3864
 3865     /* Make sure descriptors are ready before ringing the doorbell */
 3866     wmb();
 3867
     	  /*
     	  		分发命令到ufs host controller，这样就进入ufs的uic层
     	  */
 3868     /* issue command to the controller */
 3869     spin_lock_irqsave(hba->host->host_lock, flags);
 3870     ufshcd_vops_setup_xfer_req(hba, tag, (lrbp->cmd ? true : false));
 3871
 3872     err = ufshcd_send_command(hba, tag);
 3873     if (err) {
 3874         spin_unlock_irqrestore(hba->host->host_lock, flags);
 3875         scsi_dma_unmap(lrbp->cmd);
 3876         lrbp->cmd = NULL;
 3877         clear_bit_unlock(tag, &hba->lrb_in_use);
 3878         ufshcd_release_all(hba);
 3879         ufshcd_vops_pm_qos_req_end(hba, cmd->request, true);
 3880         ufshcd_vops_crypto_engine_cfg_end(hba, lrbp, cmd->request);
 3881         dev_err(hba->dev, "%s: failed sending command, %d\n",
 3882                             __func__, err);
 3883         err = DID_ERROR;
 3884         goto out;
 3885     }
 3886
 3887 out_unlock:
 3888     spin_unlock_irqrestore(hba->host->host_lock, flags);
 3889 out:
 3890     if (has_read_lock)
 3891         ufshcd_put_read_lock(hba);
 3892     return err;
 3893 }
```

