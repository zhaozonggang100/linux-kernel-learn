### 1、概述

### 2、定义

```c
// file：include/scsi/scsi_host.h

 50 struct scsi_host_template {
 51     struct module *module;	// 指向实现了这个host模板的的低层驱动模块指针
 52     const char *name;	// scsi HBA驱动的名字
 53
 54     /*
 55      * Used to initialize old-style drivers.  For new-style drivers
 56      * just perform all work in your module initialization function.
 57      *
 58      * Status:  OBSOLETE
 59      */
 60     int (* detect)(struct scsi_host_template *);
 61
 62     /*
 63      * Used as unload callback for hosts with old-style drivers.
 64      *
 65      * Status: OBSOLETE
 66      */
 67     int (* release)(struct Scsi_Host *);
 68
 69     /*
 70      * The info function will return whatever useful information the
 71      * developer sees fit.  If not provided, then the name field will
 72      * be used instead.
 73      *
 74      * Status: OPTIONAL
 75      */
 76     const char *(* info)(struct Scsi_Host *);
 77
 78     /*
 79      * Ioctl interface
 80      *
 81      * Status: OPTIONAL
 82      */
 83     int (* ioctl)(struct scsi_device *dev, int cmd, void __user *arg);
 84
 85
 86 #ifdef CONFIG_COMPAT
 87     /*
 88      * Compat handler. Handle 32bit ABI.
 89      * When unknown ioctl is passed return -ENOIOCTLCMD.
 90      *
 91      * Status: OPTIONAL
 92      */
  93     int (* compat_ioctl)(struct scsi_device *dev, int cmd, void __user *arg);
 94 #endif
 95
 96     /*
 97      * The queuecommand function is used to queue up a scsi
 98      * command block to the LLDD.  When the driver finished
 99      * processing the command the done callback is invoked.
100      *
101      * If queuecommand returns 0, then the HBA has accepted the
102      * command.  The done() function must be called on the command
103      * when the driver has finished with it. (you may call done on the
104      * command before queuecommand returns, but in this case you
105      * *must* return 0 from queuecommand).
106      *
107      * Queuecommand may also reject the command, in which case it may
108      * not touch the command and must not call done() for it.
109      *
110      * There are two possible rejection returns:
111      *
112      *   SCSI_MLQUEUE_DEVICE_BUSY: Block this device temporarily, but
113      *   allow commands to other devices serviced by this host.
114      *
115      *   SCSI_MLQUEUE_HOST_BUSY: Block all devices served by this
116      *   host temporarily.
117      *
118          * For compatibility, any other non-zero return is treated the
119          * same as SCSI_MLQUEUE_HOST_BUSY.
120      *
121      * NOTE: "temporarily" means either until the next command for#
122      * this device/host completes, or a period of time determined by
123      * I/O pressure in the system if there are no other outstanding
124      * commands.
125      *
126      * STATUS: REQUIRED
127      */
128     int (* queuecommand)(struct Scsi_Host *, struct scsi_cmnd *);
129
130     /*
131      * This is an error handling strategy routine.  You don't need to
132      * define one of these if you don't want to - there is a default
133      * routine that is present that should work in most cases.  For those
134      * driver authors that have the inclination and ability to write their
135      * own strategy routine, this is where it is specified.  Note - the
136      * strategy routine is *ALWAYS* run in the context of the kernel eh
137      * thread.  Thus you are guaranteed to *NOT* be in an interrupt
138      * handler when you execute this, and you are also guaranteed to
139      * *NOT* have any other commands being queued while you are in the
140      * strategy routine. When you return from this function, operations
141      * return to normal.
142      *
143      * See scsi_error.c scsi_unjam_host for additional comments about
144      * what this function should and should not be attempting to do.
145      *
146      * Status: REQUIRED (at least one of them)
147      */
148     int (* eh_abort_handler)(struct scsi_cmnd *);
149     int (* eh_device_reset_handler)(struct scsi_cmnd *);
150     int (* eh_target_reset_handler)(struct scsi_cmnd *);
151     int (* eh_bus_reset_handler)(struct scsi_cmnd *);
152     int (* eh_host_reset_handler)(struct scsi_cmnd *);
153
154     /*
155      * Before the mid layer attempts to scan for a new device where none
156      * currently exists, it will call this entry in your driver.  Should
157      * your driver need to allocate any structs or perform any other init
158      * items in order to send commands to a currently unused target/lun
159      * combo, then this is where you can perform those allocations.  This
160      * is specifically so that drivers won't have to perform any kind of
161      * "is this a new device" checks in their queuecommand routine,
162      * thereby making the hot path a bit quicker.
163      *
164      * Return values: 0 on success, non-0 on failure
165      *
166      * Deallocation:  If we didn't find any devices at this ID, you will
167      * get an immediate call to slave_destroy().  If we find something
168      * here then you will get a call to slave_configure(), then the
169      * device will be used for however long it is kept around, then when
170      * the device is removed from the system (or * possibly at reboot
171      * time), you will then get a call to slave_destroy().  This is
172      * assuming you implement slave_configure and slave_destroy.
173      * However, if you allocate memory and hang it off the device struct,
174      * then you must implement the slave_destroy() routine at a minimum
175      * in order to avoid leaking memory
176      * each time a device is tore down.
177      *
178      * Status: OPTIONAL
179      */
180     int (* slave_alloc)(struct scsi_device *);
181
182     /*
183      * Once the device has responded to an INQUIRY and we know the
184      * device is online, we call into the low level driver with the
185      * struct scsi_device *.  If the low level device driver implements
186      * this function, it *must* perform the task of setting the queue
187      * depth on the device.  All other tasks are optional and depend
188      * on what the driver supports and various implementation details.
189      *
190      * Things currently recommended to be handled at this time include:
191      *
192      * 1.  Setting the device queue depth.  Proper setting of this is
193      *     described in the comments for scsi_change_queue_depth.
194      * 2.  Determining if the device supports the various synchronous
195      *     negotiation protocols.  The device struct will already have
196      *     responded to INQUIRY and the results of the standard items
197      *     will have been shoved into the various device flag bits, eg.
198      *     device->sdtr will be true if the device supports SDTR messages.
199      * 3.  Allocating command structs that the device will need.
200      * 4.  Setting the default timeout on this device (if needed).
201      * 5.  Anything else the low level driver might want to do on a device
202      *     specific setup basis...
203      * 6.  Return 0 on success, non-0 on error.  The device will be marked
204      *     as offline on error so that no access will occur.  If you return
205      *     non-0, your slave_destroy routine will never get called for this
206      *     device, so don't leave any loose memory hanging around, clean
207      *     up after yourself before returning non-0
208      *
209      * Status: OPTIONAL
210      */
211     int (* slave_configure)(struct scsi_device *);
212
213     /*
214      * Immediately prior to deallocating the device and after all activity
215      * has ceased the mid layer calls this point so that the low level
216      * driver may completely detach itself from the scsi device and vice
217      * versa.  The low level driver is responsible for freeing any memory
218      * it allocated in the slave_alloc or slave_configure calls.
219      *
220      * Status: OPTIONAL
221      */
222     void (* slave_destroy)(struct scsi_device *);
223
224     /*
225      * Before the mid layer attempts to scan for a new device attached
226      * to a target where no target currently exists, it will call this
227      * entry in your driver.  Should your driver need to allocate any
228      * structs or perform any other init items in order to send commands
229      * to a currently unused target, then this is where you can perform
230      * those allocations.
231      *
232      * Return values: 0 on success, non-0 on failure
233      *
234      * Status: OPTIONAL
235      */
236     int (* target_alloc)(struct scsi_target *);
237
238     /*
239      * Immediately prior to deallocating the target structure, and
240      * after all activity to attached scsi devices has ceased, the
241      * midlayer calls this point so that the driver may deallocate
242      * and terminate any references to the target.
243      *
244      * Status: OPTIONAL
245      */
246     void (* target_destroy)(struct scsi_target *);
247
248     /*
249      * If a host has the ability to discover targets on its own instead
250      * of scanning the entire bus, it can fill in this function and
251      * call scsi_scan_host().  This function will be called periodically
252      * until it returns 1 with the scsi_host and the elapsed time of
253      * the scan in jiffies.
254      *
255      * Status: OPTIONAL
256      */
257     int (* scan_finished)(struct Scsi_Host *, unsigned long);
258
259     /*
260      * If the host wants to be called before the scan starts, but
261      * after the midlayer has set up ready for the scan, it can fill
262      * in this function.
263      *
264      * Status: OPTIONAL
265      */
266     void (* scan_start)(struct Scsi_Host *);
267
268     /*
269      * Fill in this function to allow the queue depth of this host
270      * to be changeable (on a per device basis).  Returns either
271      * the current queue depth setting (may be different from what
272      * was passed in) or an error.  An error should only be
273      * returned if the requested depth is legal but the driver was
274      * unable to set it.  If the requested depth is illegal, the
275      * driver should set and return the closest legal queue depth.
276      *
277      * Status: OPTIONAL
278      */
279     int (* change_queue_depth)(struct scsi_device *, int);
280
281     /*
282      * This functions lets the driver expose the queue mapping
283      * to the block layer.
284      *
285      * Status: OPTIONAL
286      */
287     int (* map_queues)(struct Scsi_Host *shost);
288
289     /*
290      * This function determines the BIOS parameters for a given
291      * harddisk.  These tend to be numbers that are made up by
292      * the host adapter.  Parameters:
293      * size, device, list (heads, sectors, cylinders)
294      *
295      * Status: OPTIONAL
296      */
297     int (* bios_param)(struct scsi_device *, struct block_device *,
298             sector_t, int []);
299
300     /*
301      * This function is called when one or more partitions on the
302      * device reach beyond the end of the device.
303      *
304      * Status: OPTIONAL
305      */
306     void (*unlock_native_capacity)(struct scsi_device *);
307
308     /*
309      * Can be used to export driver statistics and other infos to the
310      * world outside the kernel ie. userspace and it also provides an
311      * interface to feed the driver with information.
312      *
313      * Status: OBSOLETE
314      */
315     int (*show_info)(struct seq_file *, struct Scsi_Host *);
316     int (*write_info)(struct Scsi_Host *, char *, int);
317
318     /*
319      * This is an optional routine that allows the transport to become
320      * involved when a scsi io timer fires. The return value tells the
321      * timer routine how to finish the io timeout handling:
322      * EH_HANDLED:      I fixed the error, please complete the command
323      * EH_RESET_TIMER:  I need more time, reset the timer and
324      *          begin counting again
325      * EH_NOT_HANDLED   Begin normal error recovery
326      *
327      * Status: OPTIONAL
328      */
329     enum blk_eh_timer_return (*eh_timed_out)(struct scsi_cmnd *);
330
331     /* This is an optional routine that allows transport to initiate
332      * LLD adapter or firmware reset using sysfs attribute.
333      *
334      * Return values: 0 on success, -ve value on failure.
335      *
336      * Status: OPTIONAL
337      */
338
339     int (*host_reset)(struct Scsi_Host *shost, int reset_type);
340 #define SCSI_ADAPTER_RESET  1
341 #define SCSI_FIRMWARE_RESET 2
342
343
344     /*
345      * Name of proc directory
346      */
347     const char *proc_name;
348
349     /*
350      * Used to store the procfs directory if a driver implements the
351      * show_info method.
352      */
353     struct proc_dir_entry *proc_dir;
354
355     /*
356      * This determines if we will use a non-interrupt driven
357      * or an interrupt driven scheme.  It is set to the maximum number
358      * of simultaneous commands a given host adapter will accept.
359      */
360     int can_queue;
361
362     /*
363      * In many instances, especially where disconnect / reconnect are
364      * supported, our host also has an ID on the SCSI bus.  If this is
365      * the case, then it must be reserved.  Please set this_id to -1 if
366      * your setup is in single initiator mode, and the host lacks an
367      * ID.
368      */
369     int this_id;
370
371     /*
372      * This determines the degree to which the host adapter is capable
373      * of scatter-gather.
374      */
375     unsigned short sg_tablesize;
376     unsigned short sg_prot_tablesize;
377
378     /*
379      * Set this if the host adapter has limitations beside segment count.
380      */
381     unsigned int max_sectors;
382
383     /*
384      * DMA scatter gather segment boundary limit. A segment crossing this
385      * boundary will be split in two.
386      */
387     unsigned long dma_boundary;
388
389     /*
390      * This specifies "machine infinity" for host templates which don't
391      * limit the transfer size.  Note this limit represents an absolute
392      * maximum, and may be over the transfer limits allowed for
393      * individual devices (e.g. 256 for SCSI-1).
394      */
395 #define SCSI_DEFAULT_MAX_SECTORS    1024
396
397     /*
398      * True if this host adapter can make good use of linked commands.
399      * This will allow more than one command to be queued to a given
400      * unit on a given host.  Set this to the maximum number of command
401      * blocks to be provided for each device.  Set this to 1 for one
402      * command block per lun, 2 for two, etc.  Do not set this to 0.
403      * You should make sure that the host adapter will do the right thing
404      * before you try setting this above 1.
405      */
406     short cmd_per_lun;
407
408     /*
409      * present contains counter indicating how many boards of this
410      * type were found when we did the scan.
411      */
412     unsigned char present;
413
414     /* If use block layer to manage tags, this is tag allocation policy */
415     int tag_alloc_policy;
416
417     /*
418      * Track QUEUE_FULL events and reduce queue depth on demand.
419      */
420     unsigned track_queue_depth:1;
421
422     /*
423      * This specifies the mode that a LLD supports.
424      */
425     unsigned supported_mode:2;
426
427     /*
428      * True if this host adapter uses unchecked DMA onto an ISA bus.
429      */
430     unsigned unchecked_isa_dma:1;
431
432     /*
433      * True if this host adapter can make good use of clustering.
434      * I originally thought that if the tablesize was large that it
435      * was a waste of CPU cycles to prepare a cluster list, but
436      * it works out that the Buslogic is faster if you use a smaller
437      * number of segments (i.e. use clustering).  I guess it is
438      * inefficient.
439      */
440     unsigned use_clustering:1;
441
442     /*
443      * True for emulated SCSI host adapters (e.g. ATAPI).
444      */
445     unsigned emulated:1;
446
447     /*
448      * True if the low-level driver performs its own reset-settle delays.
449      */
450     unsigned skip_settle_delay:1;
451
452     /* True if the controller does not support WRITE SAME */
453     unsigned no_write_same:1;
454
455     /* True if the low-level driver supports blk-mq only */
456     unsigned force_blk_mq:1;
457
458     /*
459      * Countdown for host blocking with no commands outstanding.
460      */
461     unsigned int max_host_blocked;
462
463     /*
464      * Default value for the blocking.  If the queue is empty,
465      * host_blocked counts down in the request_fn until it restarts
466      * host operations as zero is reached.
467      *
468      * FIXME: This should probably be a value in the template
469      */
470 #define SCSI_DEFAULT_HOST_BLOCKED   7
471
472     /*
473      * Pointer to the sysfs class properties for this host, NULL terminated.
474      */
475     struct device_attribute **shost_attrs;
476
477     /*
478      * Pointer to the SCSI device properties for this host, NULL terminated.
479      */
480     struct device_attribute **sdev_attrs;
481
482     /*
483      * List of hosts per template.
484      *
485      * This is only for use by scsi_module.c for legacy templates.
486      * For these access to it is synchronized implicitly by
487      * module_init/module_exit.
488      */
489     struct list_head legacy_hosts;
490
491     /*
492      * Vendor Identifier associated with the host
493      *
494      * Note: When specifying vendor_id, be sure to read the
495      *   Vendor Type and ID formatting requirements specified in
496      *   scsi_netlink.h
497      */
498     u64 vendor_id;
499
500     /*
501      * Additional per-command data allocated for the driver.
502      */
503     unsigned int cmd_size;
504     struct scsi_host_cmd_pool *cmd_pool;
505 };
```

