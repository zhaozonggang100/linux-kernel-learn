```c
// file：include/scsi/scsi_device.h

96 struct scsi_device {
 97     struct Scsi_Host *host;	//指向主机适配器
 98     struct request_queue *request_queue;	//指向请求队列
 99
100     /* the next two are protected by the host->host_lock */
101     struct list_head    siblings;   /* list of all devices on this host */ //链入到主机适配器的设备链表
102     struct list_head    same_target_siblings; /* just the devices sharing same target id */ //链入目标节点的设备链表
103
104     atomic_t device_busy;       /* commands actually active on LLDD */ //已经派发的命令数
105     atomic_t device_blocked;    /* Device returned QUEUE_FULL. */
106
107     spinlock_t list_lock;
108     struct list_head cmd_list;  /* queue of in use SCSI Command structures */
109     struct list_head starved_entry;
110     unsigned short queue_depth; /* How deep of a queue we want */
111     unsigned short max_queue_depth; /* max queue depth */
112     unsigned short last_queue_full_depth; /* These two are used by */
113     unsigned short last_queue_full_count; /* scsi_track_queue_full() */
114     unsigned long last_queue_full_time; /* last queue full time */
115     unsigned long queue_ramp_up_period; /* ramp up period in jiffies */
116 #define SCSI_DEFAULT_RAMP_UP_PERIOD (120 * HZ)
117
118     unsigned long last_queue_ramp_up;   /* last queue ramp up time */
119
120     unsigned int id, channel;
121     u64 lun;
122     unsigned int manufacturer;  /* Manufacturer of device, for using
123                      * vendor-specific cmd's */
124     unsigned sector_size;   /* size in bytes */
125
126     void *hostdata;     /* available to low-level driver */
127     unsigned char type;
128     char scsi_level;
129     char inq_periph_qual;   /* PQ from INQUIRY data */
130     struct mutex inquiry_mutex;
131     unsigned char inquiry_len;  /* valid bytes in 'inquiry' */
132     unsigned char * inquiry;    /* INQUIRY response data */
133     const char * vendor;        /* [back_compat] point into 'inquiry' ... */
134     const char * model;     /* ... after scan; point to static string */
135     const char * rev;       /* ... "nullnullnullnull" before scan */
136
137 #define SCSI_VPD_PG_LEN                255
138     struct scsi_vpd __rcu *vpd_pg83;
139     struct scsi_vpd __rcu *vpd_pg80;
140     unsigned char current_tag;  /* current tag */
141     struct scsi_target      *sdev_target;   /* used only for single_lun */
142
143     unsigned int    sdev_bflags; /* black/white flags as also found in
144                  * scsi_devinfo.[hc]. For now used only to
145                  * pass settings from slave_alloc to scsi
146                  * core. */
147     unsigned int eh_timeout; /* Error handling timeout */
148     unsigned removable:1;
149     unsigned changed:1; /* Data invalid due to media change */
150     unsigned busy:1;    /* Used to prevent races */
151     unsigned lockable:1;    /* Able to prevent media removal */
152     unsigned locked:1;      /* Media removal disabled */
153     unsigned borken:1;  /* Tell the Seagate driver to be
154                  * painfully slow on this device */
155     unsigned disconnect:1;  /* can disconnect */
156     unsigned soft_reset:1;  /* Uses soft reset option */
157     unsigned sdtr:1;    /* Device supports SDTR messages */
158     unsigned wdtr:1;    /* Device supports WDTR messages */
159     unsigned ppr:1;     /* Device supports PPR messages */
160     unsigned tagged_supported:1;    /* Supports SCSI-II tagged queuing */
161     unsigned simple_tags:1; /* simple queue tag messages are enabled */
162     unsigned was_reset:1;   /* There was a bus reset on the bus for
163                  * this device */
164     unsigned expecting_cc_ua:1; /* Expecting a CHECK_CONDITION/UNIT_ATTN
165                      * because we did a bus reset. */
166     unsigned use_10_for_rw:1; /* first try 10-byte read / write */
167     unsigned use_10_for_ms:1; /* first try 10-byte mode sense/select */
168     unsigned no_report_opcodes:1;   /* no REPORT SUPPORTED OPERATION CODES */
169     unsigned no_write_same:1;   /* no WRITE SAME command */
170     unsigned use_16_for_rw:1; /* Use read/write(16) over read/write(10) */
171     unsigned skip_ms_page_8:1;  /* do not use MODE SENSE page 0x08 */
172     unsigned skip_ms_page_3f:1; /* do not use MODE SENSE page 0x3f */
173     unsigned skip_vpd_pages:1;  /* do not read VPD pages */
174     unsigned try_vpd_pages:1;   /* attempt to read VPD pages */
175     unsigned use_192_bytes_for_3f:1; /* ask for 192 bytes from page 0x3f */
176     unsigned no_start_on_add:1; /* do not issue start on add */
177     unsigned allow_restart:1; /* issue START_UNIT in error handler */
178     unsigned manage_start_stop:1;   /* Let HLD (sd) manage start/stop */
179     unsigned start_stop_pwr_cond:1; /* Set power cond. in START_STOP_UNIT */
180     unsigned no_uld_attach:1; /* disable connecting to upper level drivers */
181     unsigned select_no_atn:1;
182     unsigned fix_capacity:1;    /* READ_CAPACITY is too high by 1 */
183     unsigned guess_capacity:1;  /* READ_CAPACITY might be too high by 1 */
184     unsigned retry_hwerror:1;   /* Retry HARDWARE_ERROR */
185     unsigned last_sector_bug:1; /* do not use multisector accesses on
186                        SD_LAST_BUGGY_SECTORS */
187     unsigned no_read_disc_info:1;   /* Avoid READ_DISC_INFO cmds */
188     unsigned no_read_capacity_16:1; /* Avoid READ_CAPACITY_16 cmds */
189     unsigned try_rc_10_first:1; /* Try READ_CAPACACITY_10 first */
190     unsigned security_supported:1;  /* Supports Security Protocols */
191     unsigned is_visible:1;  /* is the device visible in sysfs */
192     unsigned wce_default_on:1;  /* Cache is ON by default */
193     unsigned no_dif:1;  /* T10 PI (DIF) should be disabled */
194     unsigned broken_fua:1;      /* Don't set FUA bit */
195     unsigned lun_in_cdb:1;      /* Store LUN bits in CDB[1] */
196     unsigned unmap_limit_for_ws:1;  /* Use the UNMAP limit for WRITE SAME */
197     unsigned use_rpm_auto:1; /* Enable runtime PM auto suspend */
198
199 #define SCSI_DEFAULT_AUTOSUSPEND_DELAY  -1
200     int autosuspend_delay;
201     /* If non-zero, use timeout (in jiffies) for all commands */
202     unsigned int timeout_override;
203
204     atomic_t disk_events_disable_depth; /* disable depth for disk events */
205
206     DECLARE_BITMAP(supported_events, SDEV_EVT_MAXBITS); /* supported events */
207     DECLARE_BITMAP(pending_events, SDEV_EVT_MAXBITS); /* pending events */
208     struct list_head event_list;    /* asserted events */
209     struct work_struct event_work;
210
211     unsigned int max_device_blocked; /* what device_blocked counts down from  */
212 #define SCSI_DEFAULT_DEVICE_BLOCKED 3
213
214     atomic_t iorequest_cnt;
215     atomic_t iodone_cnt;
216     atomic_t ioerr_cnt;
217
218     struct device       sdev_gendev,
219                 sdev_dev;
220
221     struct execute_work ew; /* used to get process context on put */
222     struct work_struct  requeue_work;
223
224     struct scsi_device_handler *handler;
225     void            *handler_data;
226
227     unsigned char       access_state;
228     struct mutex        state_mutex;
229     enum scsi_device_state sdev_state;
230     unsigned long       sdev_data[0];
231 } __attribute__((aligned(sizeof(unsigned long))));
```

