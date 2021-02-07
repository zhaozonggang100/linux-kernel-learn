**ref** 
```
https://zhuanlan.zhihu.com/p/30691789（旧版本Android）
https://blog.csdn.net/yiranfeng/article/details/105181297（基于Android10）
```

---

### 1、概述

1、从Android 8.0 开始binder被拆分为：

**binder**：System分区 进程间通信

**hwbinder**：System/Vendor分区进程间通信

**vndbinder**：Vendor分区进程间通信

2、关键术语  

**bpbinder**：binder代理，client端访问server端binder实体的引用

**bbinder**：binder实体，server端binder的表示结构

**ServiceManager**：binder架构的native层的0号binder服务，用于server注册服务，client查询服务等管理工作

3、binder内核设备文件
```bash
crw-rw-rw- 1 root            root             10,  50 2021-02-03 17:48 binder
crw-rw-rw- 1 root            root             10,  49 2021-02-03 17:48 hwbinder
crw-rw-rw- 1 root            root             10,  48 2021-02-03 17:48 vndbinder
```


### 2、结构体

- 1、binder进程

每个进程调用open()打开binder驱动都会创建该结构体，用于管理IPC所需的各种信息

```c++
// file: drivers/android/binder.c
/**
 * struct binder_proc - binder process bookkeeping
 * @proc_node:            element for binder_procs list
 * @threads:              rbtree of binder_threads in this proc
 *                        (protected by @inner_lock)
 * @nodes:                rbtree of binder nodes associated with
 *                        this proc ordered by node->ptr
 *                        (protected by @inner_lock)
 * @refs_by_desc:         rbtree of refs ordered by ref->desc
 *                        (protected by @outer_lock)
 * @refs_by_node:         rbtree of refs ordered by ref->node
 *                        (protected by @outer_lock)
 * @waiting_threads:      threads currently waiting for proc work
 *                        (protected by @inner_lock)
 * @pid                   PID of group_leader of process
 *                        (invariant after initialized)
 * @tsk                   task_struct for group_leader of process
 *                        (invariant after initialized)
 * @deferred_work_node:   element for binder_deferred_list
 *                        (protected by binder_deferred_lock)
 * @deferred_work:        bitmap of deferred work to perform
 *                        (protected by binder_deferred_lock)
 * @is_dead:              process is dead and awaiting free
 *                        when outstanding transactions are cleaned up
 *                        (protected by @inner_lock)
 * @todo:                 list of work for this process
 *                        (protected by @inner_lock)
 * @stats:                per-process binder statistics
 *                        (atomics, no lock needed)
 * @delivered_death:      list of delivered death notification
 *                        (protected by @inner_lock)
 * @max_threads:          cap on number of binder threads
 *                        (protected by @inner_lock)
 * @requested_threads:    number of binder threads requested but not
 *                        yet started. In current implementation, can
 *                        only be 0 or 1.
 *                        (protected by @inner_lock)
 * @requested_threads_started: number binder threads started
 *                        (protected by @inner_lock)
 * @tmp_ref:              temporary reference to indicate proc is in use
 *                        (protected by @inner_lock)
 * @requested_threads_started: number binder threads started
 *                        (protected by @inner_lock)
 * @tmp_ref:              temporary reference to indicate proc is in use
 *                        (protected by @inner_lock)
 * @default_priority:     default scheduler priority
 *                        (invariant after initialized)
 * @debugfs_entry:        debugfs node
 * @alloc:                binder allocator bookkeeping
 * @context:              binder_context for this proc
 *                        (invariant after initialized)
 * @inner_lock:           can nest under outer_lock and/or node lock
 * @outer_lock:           no nesting under innor or node lock
 *                        Lock order: 1) outer, 2) node, 3) inner
 *
 * Bookkeeping structure for binder processes
 */
struct binder_proc {
        struct hlist_node proc_node;                    // 将所有binder进程组织到全局的哈希链表中
        struct rb_root threads;                         // binder_thread红黑树的根节点
        struct rb_root nodes;                           // binder_node红黑树的根节点
        struct rb_root refs_by_desc;                    // binder_ref红黑树的根节点(以handle为key)
        struct rb_root refs_by_node;                    // binder_ref红黑树的根节点（以ptr为key）
        struct list_head waiting_threads;
        int pid;                                        // 相应进程id
        struct task_struct *tsk;                        // 相应进程的task结构体
        struct hlist_node deferred_work_node;
        int deferred_work;
        bool is_dead;

        struct list_head todo;
        struct binder_stats stats;
        struct list_head delivered_death;
        int max_threads;
        int requested_threads;
        int requested_threads_started;
        int tmp_ref;
        struct binder_priority default_priority;
        struct dentry *debugfs_entry;
        struct binder_alloc alloc;
        struct binder_context *context;
        spinlock_t inner_lock;
        spinlock_t outer_lock;
};
```

- 2、binder线程  

对应于上层的binder线程，当前binder操作所在的线程

```c++
/**
 * struct binder_thread - binder thread bookkeeping
 * @proc:                 binder process for this thread
 *                        (invariant after initialization)
 * @rb_node:              element for proc->threads rbtree
 *                        (protected by @proc->inner_lock)
 * @waiting_thread_node:  element for @proc->waiting_threads list
 *                        (protected by @proc->inner_lock)
 * @pid:                  PID for this thread
 *                        (invariant after initialization)
 * @looper:               bitmap of looping state
 *                        (only accessed by this thread)
 * @looper_needs_return:  looping thread needs to exit driver
 *                        (no lock needed)
 * @transaction_stack:    stack of in-progress transactions for this thread
 *                        (protected by @proc->inner_lock)
 * @todo:                 list of work to do for this thread
 *                        (protected by @proc->inner_lock)
 * @process_todo:         whether work in @todo should be processed
 *                        (protected by @proc->inner_lock)
 * @return_error:         transaction errors reported by this thread
 *                        (only accessed by this thread)
 * @reply_error:          transaction errors reported by target thread
 *                        (protected by @proc->inner_lock)
 * @wait:                 wait queue for thread work
 * @stats:                per-thread statistics
 *                        (atomics, no lock needed)
 * @tmp_ref:              temporary reference to indicate thread is in use
 *                        (atomic since @proc->inner_lock cannot
 *                        always be acquired)
 * @is_dead:              thread is dead and awaiting free
 *                        when outstanding transactions are cleaned up
 *                        (protected by @proc->inner_lock)
 * @task:                 struct task_struct for this thread
 *      
 * Bookkeeping structure for binder threads.
 */
struct binder_thread {
        // 线程所属的进程
        struct binder_proc *proc;
        struct rb_node rb_node;
        struct list_head waiting_thread_node;
        // 线程pid
        int pid;
        // looper的状态
        /*
                enum {
                        BINDER_LOOPER_STATE_REGISTERED  = 0x01, // 创建注册线程BC_REGISTER_LOOPER
                        BINDER_LOOPER_STATE_ENTERED     = 0x02, // 创建主线程BC_ENTER_LOOPER
                        BINDER_LOOPER_STATE_EXITED      = 0x04, // 已退出
                        BINDER_LOOPER_STATE_INVALID     = 0x08, // 非法
                        BINDER_LOOPER_STATE_WAITING     = 0x10, // 等待中
                        BINDER_LOOPER_STATE_NEED_RETURN = 0x20, // 需要返回
                };
        */
        int looper;              /* only modified by this thread */
        bool looper_need_return; /* can be written by other thread */
        // 线程正在处理的事务
        struct binder_transaction *transaction_stack;
        // 将要处理的链表
        struct list_head todo;
        bool process_todo;
        // write失败后，返回的错误码
        struct binder_error return_error;
        struct binder_error reply_error;
        wait_queue_head_t wait;
        // binder线程的统计信息
        struct binder_stats stats;
        atomic_t tmp_ref;
        bool is_dead;
        struct task_struct *task;
};
```

- 3、binder实体  

1、对应于BBinder对象，记录BBinder的进程、指针、引用计数等  
2、当Binder对象已销毁，但还存在该Binder节点引用，则采用dead_node，并加入到全局列表binder_dead_nodes；否则使用rb_node节点。

```c++
/**
 * struct binder_node - binder node bookkeeping
 * @debug_id:             unique ID for debugging
 *                        (invariant after initialized)
 * @lock:                 lock for node fields
 * @work:                 worklist element for node work
 *                        (protected by @proc->inner_lock)
 * @rb_node:              element for proc->nodes tree
 *                        (protected by @proc->inner_lock)
 * @dead_node:            element for binder_dead_nodes list
 *                        (protected by binder_dead_nodes_lock)
 * @proc:                 binder_proc that owns this node
 *                        (invariant after initialized)
 * @refs:                 list of references on this node
 *                        (protected by @lock)
 * @internal_strong_refs: used to take strong references when
 *                        initiating a transaction
 *                        (protected by @proc->inner_lock if @proc
 *                        and by @lock)
 * @local_weak_refs:      weak user refs from local process
 *                        (protected by @proc->inner_lock if @proc
 *                        and by @lock)
 * @local_strong_refs:    strong user refs from local process
 *                        (protected by @proc->inner_lock if @proc
 *                        and by @lock)
 * @tmp_refs:             temporary kernel refs
 *                        (protected by @proc->inner_lock while @proc
 *                        is valid, and by binder_dead_nodes_lock
 *                        if @proc is NULL. During inc/dec and node release
 *                        it is also protected by @lock to provide safety
 *                        as the node dies and @proc becomes NULL)
 * @ptr:                  userspace pointer for node
 *                        (invariant, no lock needed)
 * @cookie:               userspace cookie for node
 *                        (invariant, no lock needed)
 * @has_strong_ref:       userspace notified of strong ref
 *                        (protected by @proc->inner_lock if @proc
 *                        and by @lock)
 * @pending_strong_ref:   userspace has acked notification of strong ref
 *                        (protected by @proc->inner_lock if @proc
 *                        and by @lock)
 * @has_weak_ref:         userspace notified of weak ref
 *                        (protected by @proc->inner_lock if @proc
 *                        and by @lock)
 * @pending_weak_ref:     userspace has acked notification of weak ref
 *                        (protected by @proc->inner_lock if @proc
 *                        and by @lock)
 * @has_async_transaction: async transaction to node in progress
  *                        (protected by @lock)
 * @sched_policy:         minimum scheduling policy for node
 *                        (invariant after initialized)
 * @accept_fds:           file descriptor operations supported for node
 *                        (invariant after initialized)
 * @min_priority:         minimum scheduling priority
 *                        (invariant after initialized)
 * @inherit_rt:           inherit RT scheduling policy from caller
 *                        (invariant after initialized)
 * @async_todo:           list of async work items
 *                        (protected by @proc->inner_lock)
 *
 * Bookkeeping structure for binder nodes.
 */
struct binder_node {
        // 节点创建时分配，具有全局唯一性，用于调试使用
        int debug_id;
        spinlock_t lock;
        struct binder_work work;
        union {
                // binder节点正常使用，union
                struct rb_node rb_node;
                // binder节点已销毁，union
                struct hlist_node dead_node;
        };
        // binder所在的进程
        struct binder_proc *proc;
        // 所有指向该节点的binder引用队列
        struct hlist_head refs;
        int internal_strong_refs;
        int local_weak_refs;
        int local_strong_refs;
        int tmp_refs;
        // 指向用户空间binder_node的指针
        binder_uintptr_t ptr;
        // 附件数据
        binder_uintptr_t cookie;
        struct {
                /*
                 * bitfield elements protected by
                 * proc inner_lock
                 */
                u8 has_strong_ref:1;
                u8 pending_strong_ref:1;
                u8 has_weak_ref:1;
                u8 pending_weak_ref:1;
        };
        struct {
                /*
                 * invariant after initialization
                 */
                u8 sched_policy:2;
                u8 inherit_rt:1;
                u8 accept_fds:1;
                u8 min_priority;
                };
        bool has_async_transaction;
        // 异步todo队列
        struct list_head async_todo;
};
```

- 4、binder引用

对应于BpBinder对象，记录BpBinder的引用计数、死亡通知、BBinder指针等

```c++
/**
 * struct binder_ref - struct to track references on nodes
 * @data:        binder_ref_data containing id, handle, and current refcounts
 * @rb_node_desc: node for lookup by @data.desc in proc's rb_tree
 * @rb_node_node: node for lookup by @node in proc's rb_tree
 * @node_entry:  list entry for node->refs list in target node
 *               (protected by @node->lock)
 * @proc:        binder_proc containing ref
 * @node:        binder_node of target node. When cleaning up a
 *               ref for deletion in binder_cleanup_ref, a non-NULL
 *               @node indicates the node must be freed
 * @death:       pointer to death notification (ref_death) if requested
 *               (protected by @node->lock)
 *
 * Structure to track references from procA to target node (on procB). This
 * structure is unsafe to access without holding @proc->outer_lock.
 */
struct binder_ref {
        /* Lookups needed: */
        /*   node + proc => ref (transaction) */
        /*   desc + proc => ref (transaction, inc/dec ref) */
        /*   node => refs + procs (proc exit) */
        struct binder_ref_data data;
        struct rb_node rb_node_desc;
        struct rb_node rb_node_node;
        struct hlist_node node_entry;
        struct binder_proc *proc;
        struct binder_node *node;
        struct binder_ref_death *death;
};
```

- 5、binder死亡引用  

记录binder死亡的引用信息

```c++
struct binder_ref_death {
        /**
         * @work: worklist element for death notifications
         *        (protected by inner_lock of the proc that
         *        this ref belongs to)
         */
        struct binder_work work;
        binder_uintptr_t cookie;
};
```

- 6、binder读写

记录buffer中读和写的数据信息

```c++
/*
 * On 64-bit platforms where user code may run in 32-bits the driver must
 * translate the buffer (and local binder) addresses appropriately.
 */

struct binder_write_read {
        binder_size_t           write_size;     /* bytes to write */
        binder_size_t           write_consumed; /* bytes consumed by driver */
        binder_uintptr_t        write_buffer;
        binder_size_t           read_size;      /* bytes to read */
        binder_size_t           read_consumed;  /* bytes consumed by driver */
        binder_uintptr_t        read_buffer;
};
```

- 7、binder事务数据

记录传输数据内容，比如发送方pid/uid，RPC数据

```c++
struct binder_transaction_data {
        /* The first two are only used for bcTRANSACTION and brTRANSACTION,
         * identifying the target and contents of the transaction.
         */
        union {
                /* target descriptor of command transaction */
                __u32   handle; 
                /* target descriptor of return transaction */
                binder_uintptr_t ptr;
        } target;
        binder_uintptr_t        cookie; /* target object cookie */
        __u32           code;           /* transaction command */
        
        /* General information about the transaction. */
        __u32           flags;
        pid_t           sender_pid;
        uid_t           sender_euid;
        binder_size_t   data_size;      /* number of bytes of data */
        binder_size_t   offsets_size;   /* number of bytes of offsets */

        /* If this transaction is inline, the data immediately
         * follows here; otherwise, it ends with a pointer to
         * the data buffer.
         */
        union {
                struct {
                        /* transaction data */
                        binder_uintptr_t        buffer;
                        /* offsets from buffer to flat_binder_object structs */
                        binder_uintptr_t        offsets;
                } ptr;
                __u8    buf[8];
        } data;
};
```

- 8、binder扁平对象

Binder对象在两个进程间传递的扁平结构

```c++
/*      
 * This is the flattened representation of a Binder object for transfer
 * between processes.  The 'offsets' supplied as part of a binder transaction
 * contains offsets into the data where these structures occur.  The Binder
 * driver takes care of re-writing the structure type and data as it moves
 * between processes.
 */     
struct flat_binder_object {
        struct binder_object_header     hdr;
        __u32                           flags;

        /* 8 bytes of data. */
        union {         
                binder_uintptr_t        binder; /* local object */
                __u32                   handle; /* remote object */
        };
        
        /* extra data associated with local object */
        binder_uintptr_t        cookie;
};
```

- 9、binder内存

调用mmap()创建用于Binder传输数据的缓存区

```c++
/**     
 * struct binder_buffer - buffer used for binder transactions
 * @entry:              entry alloc->buffers
 * @rb_node:            node for allocated_buffers/free_buffers rb trees
 * @free:               true if buffer is free
 * @allow_user_free:    describe the second member of struct blah,
 * @async_transaction:  describe the second member of struct blah,
 * @debug_id:           describe the second member of struct blah,
 * @transaction:        describe the second member of struct blah,
 * @target_node:        describe the second member of struct blah,
 * @data_size:          describe the second member of struct blah,
 * @offsets_size:       describe the second member of struct blah,
 * @extra_buffers_size: describe the second member of struct blah,
 * @data:i              describe the second member of struct blah,
 *
 * Bookkeeping structure for binder transaction buffers
 */
struct binder_buffer {
        struct list_head entry; /* free and allocated entries by address */
        struct rb_node rb_node; /* free entry by size or allocated entry */
                                /* by address */
        unsigned free:1;
        unsigned allow_user_free:1;
        unsigned async_transaction:1;
        unsigned free_in_progress:1;
        unsigned debug_id:28;

        struct binder_transaction *transaction;

        struct binder_node *target_node;
        size_t data_size;
        size_t offsets_size;
        size_t extra_buffers_size;
        void *data;
};
```

- 10、binder事务

记录传输事务的发送方和接收方线程、进程等

```c++
struct binder_transaction {
        int debug_id;
        struct binder_work work;
        struct binder_thread *from;
        struct binder_transaction *from_parent;
        struct binder_proc *to_proc;
        struct binder_thread *to_thread;
        struct binder_transaction *to_parent;
        unsigned need_reply:1;
        /* unsigned is_dead:1; */       /* not used at the moment */

        struct binder_buffer *buffer;
        unsigned int    code;
        unsigned int    flags;
        struct binder_priority  priority;
        struct binder_priority  saved_priority;
        bool    set_priority_called;
        kuid_t  sender_euid;
        /**
         * @lock:  protects @from, @to_proc, and @to_thread
         *
         * @from, @to_proc, and @to_thread can be set to NULL
         * during thread teardown
         */
        spinlock_t lock;
};
```

- 11、binder工作

binder的工作类型

```c++
/**
 * struct binder_work - work enqueued on a worklist
 * @entry:             node enqueued on list
 * @type:              type of work to be performed
 *      
 * There are separate work lists for proc, thread, and node (async).
 */
struct binder_work {
        struct list_head entry;

        enum {
                BINDER_WORK_TRANSACTION = 1,
                BINDER_WORK_TRANSACTION_COMPLETE,
                BINDER_WORK_RETURN_ERROR,
                BINDER_WORK_NODE,
                BINDER_WORK_DEAD_BINDER,
                BINDER_WORK_DEAD_BINDER_AND_CLEAR,
                BINDER_WORK_CLEAR_DEATH_NOTIFICATION,
        } type; 
};
```

- 12、binder状态

```c++
/**
 * struct binder_work - work enqueued on a worklist
 * @entry:             node enqueued on list
 * @type:              type of work to be performed
 *      
 * There are separate work lists for proc, thread, and node (async).
 */
struct binder_work {
        struct list_head entry;

        enum {
                BINDER_WORK_TRANSACTION = 1,
                BINDER_WORK_TRANSACTION_COMPLETE,
                BINDER_WORK_RETURN_ERROR,
                BINDER_WORK_NODE,
                BINDER_WORK_DEAD_BINDER,
                BINDER_WORK_DEAD_BINDER_AND_CLEAR,
                BINDER_WORK_CLEAR_DEATH_NOTIFICATION,
        } type; 
};
```