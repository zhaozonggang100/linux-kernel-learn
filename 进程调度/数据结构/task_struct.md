```c
1638 struct task_struct {
1639 #ifdef CONFIG_THREAD_INFO_IN_TASK		// arm64 = y，那么thread_info和进程内核栈不是共用体
1640     /*
1641      * For reasons of header soup (see current_thread_info()), this
1642      * must be the first element of task_struct.
1643      */
1644     struct thread_info thread_info;
1645 #endif
1646     volatile long state;    /* -1 unrunnable, 0 runnable, >0 stopped */
		 // 进程内核栈
1647     void *stack;			
1648     atomic_t usage;
1649     unsigned int flags; /* per process flags, defined below */
1650     unsigned int ptrace;
1651 
1652 #ifdef CONFIG_SMP      
1653     struct llist_node wake_entry;
1654     int on_cpu;
1655 #ifdef CONFIG_THREAD_INFO_IN_TASK  
1656     unsigned int cpu;   /* current CPU */
1657 #endif
1658     unsigned int wakee_flips;
1659     unsigned long wakee_flip_decay_ts;
1660     struct task_struct *last_wakee;
1661 
1662     int wake_cpu;
1663 #endif
1664     int on_rq;
1665 
1666     int prio, static_prio, normal_prio;
1667     unsigned int rt_priority;
1668     const struct sched_class *sched_class;
1669     struct sched_entity se;
1670     struct sched_rt_entity rt;
1671 #ifdef CONFIG_SCHED_HMP
1672     struct ravg ravg;
1673     /*
1674      * 'init_load_pct' represents the initial task load assigned to children
1675      * of this task
1676      */
1677     u32 init_load_pct;
1678     u64 last_wake_ts;
1679     u64 last_switch_out_ts;
1680     u64 last_cpu_selected_ts;
1681     struct related_thread_group *grp;
1682     struct list_head grp_list;
1683     u64 cpu_cycles;
1684     u64 last_sleep_ts;
1685 #endif
1686 #ifdef CONFIG_CGROUP_SCHED
1687     struct task_group *sched_task_group;
1688 #endif
1689     struct sched_dl_entity dl;
1690 
1691 #ifdef CONFIG_PREEMPT_NOTIFIERS
1692     /* list of struct preempt_notifier: */
1693     struct hlist_head preempt_notifiers;
1694 #endif
1695 
1696 #ifdef CONFIG_BLK_DEV_IO_TRACE
1697     unsigned int btrace_seq;
1698 #endif
1699 
1700     unsigned int policy;
1701     int nr_cpus_allowed;
1702     cpumask_t cpus_allowed;
1703 
1704 #ifdef CONFIG_PREEMPT_RCU
1705     int rcu_read_lock_nesting;
1706     union rcu_special rcu_read_unlock_special;
1707     struct list_head rcu_node_entry;
1708     struct rcu_node *rcu_blocked_node;
1709 #endif /* #ifdef CONFIG_PREEMPT_RCU */
1710 #ifdef CONFIG_TASKS_RCU
1711     unsigned long rcu_tasks_nvcsw;
1712     bool rcu_tasks_holdout;
1713     struct list_head rcu_tasks_holdout_list;
1714     int rcu_tasks_idle_cpu;
1715 #endif /* #ifdef CONFIG_TASKS_RCU */
1716 
1717 #ifdef CONFIG_SCHED_INFO
1718     struct sched_info sched_info;
1719 #endif
1720 
1721     struct list_head tasks;
1722 #ifdef CONFIG_SMP
1723     struct plist_node pushable_tasks;
1724     struct rb_node pushable_dl_tasks;
1725 #endif
1726 
1727     struct mm_struct *mm, *active_mm;
1728     /* per-thread vma caching */
1729     u32 vmacache_seqnum;
1730     struct vm_area_struct *vmacache[VMACACHE_SIZE];
1731 #if defined(SPLIT_RSS_COUNTING)
1732     struct task_rss_stat    rss_stat;
1733 #endif
1734 /* task state */
1735     int exit_state;
1736     int exit_code, exit_signal;
1737     int pdeath_signal;  /*  The signal sent when the parent dies  */
1738     unsigned long jobctl;   /* JOBCTL_*, siglock protected */
1739 
1740     /* Used for emulating ABI behavior of previous Linux versions */
1741     unsigned int personality;
1742 
1743     /* scheduler bits, serialized by scheduler locks */
1744     unsigned sched_reset_on_fork:1;
1745     unsigned sched_contributes_to_load:1;
1746     unsigned sched_migrated:1;
1747     unsigned :0; /* force alignment to the next boundary */
1748 
1749     /* unserialized, strictly 'current' */
1750     unsigned in_execve:1; /* bit to tell LSMs we're in execve */
1751     unsigned in_iowait:1;
1752 #ifdef CONFIG_MEMCG
1753     unsigned memcg_may_oom:1;
1754 #endif
1755 #ifdef CONFIG_MEMCG_KMEM
1756     unsigned memcg_kmem_skip_account:1;
1757 #endif
1758 #ifdef CONFIG_COMPAT_BRK
1759     unsigned brk_randomized:1;
1760 #endif
1761 #ifdef CONFIG_CGROUPS
1762     /* disallow userland-initiated cgroup migration */
1763     unsigned no_cgroup_migration:1;
1764 #endif
1765 
1766     unsigned long atomic_flags; /* Flags needing atomic access. */
1767 
1768     struct restart_block restart_block;
1769 
1770     pid_t pid;
1771     pid_t tgid;
1772 
1773 #ifdef CONFIG_CC_STACKPROTECTOR
1774     /* Canary value for the -fstack-protector gcc feature */
1775     unsigned long stack_canary;
1776 #endif
1777     /*
1778      * pointers to (original) parent process, youngest child, younger sibling,
1779      * older sibling, respectively.  (p->father can be replaced with
1780      * p->real_parent->pid)
1781      */
1782     struct task_struct __rcu *real_parent; /* real parent process */
1783     struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */
1784     /*
1785      * children/sibling forms the list of my natural children
1786      */
1787     struct list_head children;  /* list of my children */
1788     struct list_head sibling;   /* linkage in my parent's children list */
1789     struct task_struct *group_leader;   /* threadgroup leader */
1790 
1791     /*
1792      * ptraced is the list of tasks this task is using ptrace on.
1793      * This includes both natural children and PTRACE_ATTACH targets.
1794      * p->ptrace_entry is p's link on the p->parent->ptraced list.
1795      */
1796     struct list_head ptraced;
1797     struct list_head ptrace_entry;
1798 
1799     /* PID/PID hash table linkage. */
1800     struct pid_link pids[PIDTYPE_MAX];
1801     struct list_head thread_group;
1802     struct list_head thread_node;
1803 
1804     struct completion *vfork_done;      /* for vfork() */
1805     int __user *set_child_tid;      /* CLONE_CHILD_SETTID */
1806     int __user *clear_child_tid;        /* CLONE_CHILD_CLEARTID */
1807 
1808     cputime_t utime, stime, utimescaled, stimescaled;
1809     cputime_t gtime;
1810 #ifdef CONFIG_CPU_FREQ_TIMES
1811     u64 *time_in_state;
1812     unsigned int max_state;
1813 #endif
1814     struct prev_cputime prev_cputime;
1815 #ifdef CONFIG_VIRT_CPU_ACCOUNTING_GEN
1816     seqlock_t vtime_seqlock;
1817     unsigned long long vtime_snap;
1818     enum {
1819         VTIME_SLEEPING = 0,
1820         VTIME_USER,
1821         VTIME_SYS,
1822     } vtime_snap_whence;
1823 #endif
1824     unsigned long nvcsw, nivcsw; /* context switch counts */
1825     u64 start_time;     /* monotonic time in nsec */
1826     u64 real_start_time;    /* boot based time in nsec */
1827 /* mm fault and swap info: this can arguably be seen as either mm-specific or thread-specific */
1828     unsigned long min_flt, maj_flt;
1829 
1830     struct task_cputime cputime_expires;
1831     struct list_head cpu_timers[3];
1832 
1833 /* process credentials */
1834     const struct cred __rcu *ptracer_cred; /* Tracer's credentials at attach */
1835     const struct cred __rcu *real_cred; /* objective and real subjective task
1836                      * credentials (COW) */
1837     const struct cred __rcu *cred;  /* effective (overridable) subjective task
1838                      * credentials (COW) */
1839     char comm[TASK_COMM_LEN]; /* executable name excluding path
1840                      - access with [gs]et_task_comm (which lock
1841                        it with task_lock())
1842                      - initialized normally by setup_new_exec */
1843 /* file system info */
1844     struct nameidata *nameidata;
1845 #ifdef CONFIG_SYSVIPC
1846 /* ipc stuff */
1847     struct sysv_sem sysvsem;
1848     struct sysv_shm sysvshm;
1849 #endif
1850 #ifdef CONFIG_DETECT_HUNG_TASK
1851 /* hung task detection */
1852     unsigned long last_switch_count;
1853 #endif
1854 /* filesystem information */
1855     struct fs_struct *fs;
1856 /* open file information */
1857     struct files_struct *files;
1858 /* namespaces */
1859     struct nsproxy *nsproxy;
1860 /* signal handlers */
1861     struct signal_struct *signal;
1862     struct sighand_struct *sighand;
1863 
1864     sigset_t blocked, real_blocked;
1865     sigset_t saved_sigmask; /* restored if set_restore_sigmask() was used */
1866     struct sigpending pending;
1867 
1868     unsigned long sas_ss_sp;
1869     size_t sas_ss_size;
1870 
1871     struct callback_head *task_works;
1872 
1873     struct audit_context *audit_context;
1874 #ifdef CONFIG_AUDITSYSCALL
1875     kuid_t loginuid;
1876     unsigned int sessionid;
1877 #endif
1878     struct seccomp seccomp;
1879 
1880 /* Thread group tracking */
1881     u32 parent_exec_id;
1882     u32 self_exec_id;
1883 /* Protection of (de-)allocation: mm, files, fs, tty, keyrings, mems_allowed,
1884  * mempolicy */
1885     spinlock_t alloc_lock;
1886 
1887     /* Protection of the PI data structures: */
1888     raw_spinlock_t pi_lock;
1889 
1890     struct wake_q_node wake_q;
1891 
1892 #ifdef CONFIG_RT_MUTEXES
1893     /* PI waiters blocked on a rt_mutex held by this task */
1894     struct rb_root pi_waiters;
1895     struct rb_node *pi_waiters_leftmost;
1896     /* Deadlock detection and priority inheritance handling */
1897     struct rt_mutex_waiter *pi_blocked_on;
1898 #endif
1899 
1900 #ifdef CONFIG_DEBUG_MUTEXES
1901     /* mutex deadlock detection */
1902     struct mutex_waiter *blocked_on;
1903 #endif
1904 #ifdef CONFIG_TRACE_IRQFLAGS
1905     unsigned int irq_events;
1906     unsigned long hardirq_enable_ip;
1907     unsigned long hardirq_disable_ip;
1908     unsigned int hardirq_enable_event;
1909     unsigned int hardirq_disable_event;
1910     int hardirqs_enabled;
1911     int hardirq_context;
1912     unsigned long softirq_disable_ip;
1913     unsigned long softirq_enable_ip;
1914     unsigned int softirq_disable_event;
1915     unsigned int softirq_enable_event;
1916     int softirqs_enabled;
1917     int softirq_context;
1918 #endif
1919 #ifdef CONFIG_LOCKDEP
1920 # define MAX_LOCK_DEPTH 48UL
1921     u64 curr_chain_key;
1922     int lockdep_depth;
1923     unsigned int lockdep_recursion;
1924     struct held_lock held_locks[MAX_LOCK_DEPTH];
1925     gfp_t lockdep_reclaim_gfp;
1926 #endif
1927 #ifdef CONFIG_UBSAN
1928     unsigned int in_ubsan;
1929 #endif
1930 
1931 /* journalling filesystem info */
1932     void *journal_info;
1933 
1934 /* stacked block device info */
1935     struct bio_list *bio_list;
1936 
1937 #ifdef CONFIG_BLOCK
1938 /* stack plugging */
1939     struct blk_plug *plug;
1940 #endif
1941 
1942 /* VM state */
1943     struct reclaim_state *reclaim_state;
1944 
1945     struct backing_dev_info *backing_dev_info;
1946 
1947     struct io_context *io_context;
1948 
1949     unsigned long ptrace_message;
1950     siginfo_t *last_siginfo; /* For ptrace use.  */
1951     struct task_io_accounting ioac;
1952 #if defined(CONFIG_TASK_XACCT)
1953     u64 acct_rss_mem1;  /* accumulated rss usage */
1954     u64 acct_vm_mem1;   /* accumulated virtual memory usage */
1955     cputime_t acct_timexpd; /* stime + utime since last update */
1956 #endif
1957 #ifdef CONFIG_CPUSETS
1958     nodemask_t mems_allowed;    /* Protected by alloc_lock */
1959     seqcount_t mems_allowed_seq;    /* Seqence no to catch updates */
1960     int cpuset_mem_spread_rotor;
1961     int cpuset_slab_spread_rotor;
1962 #endif
1963 #ifdef CONFIG_CGROUPS
1964     /* Control Group info protected by css_set_lock */
1965     struct css_set __rcu *cgroups;
1966     /* cg_list protected by css_set_lock and tsk->alloc_lock */
1967     struct list_head cg_list;
1968 #endif
1969 #ifdef CONFIG_FUTEX
1970     struct robust_list_head __user *robust_list;
1971 #ifdef CONFIG_COMPAT
1972     struct compat_robust_list_head __user *compat_robust_list;
1973 #endif
1974     struct list_head pi_state_list;
1975     struct futex_pi_state *pi_state_cache;
1976 #endif
1977 #ifdef CONFIG_PERF_EVENTS
1978     struct perf_event_context *perf_event_ctxp[perf_nr_task_contexts];
1979     struct mutex perf_event_mutex;
1980     struct list_head perf_event_list;
1981 #endif
1982 #ifdef CONFIG_DEBUG_PREEMPT
1983     unsigned long preempt_disable_ip;
1984 #endif
1985 #ifdef CONFIG_NUMA
1986     struct mempolicy *mempolicy;    /* Protected by alloc_lock */
1987     short il_next;
1988     short pref_node_fork;
1989 #endif
1990 #ifdef CONFIG_NUMA_BALANCING
1991     int numa_scan_seq;
1992     unsigned int numa_scan_period;
1993     unsigned int numa_scan_period_max;
1994     int numa_preferred_nid;
1995     unsigned long numa_migrate_retry;
1996     u64 node_stamp;         /* migration stamp  */
1997     u64 last_task_numa_placement;
1998     u64 last_sum_exec_runtime;
1999     struct callback_head numa_work;
2000 
2001     struct list_head numa_entry;
2002     struct numa_group *numa_group;
2003 
2004     /*
2005      * numa_faults is an array split into four regions:
2006      * faults_memory, faults_cpu, faults_memory_buffer, faults_cpu_buffer
2007      * in this precise order.
2008      *
2009      * faults_memory: Exponential decaying average of faults on a per-node
2010      * basis. Scheduling placement decisions are made based on these
2011      * counts. The values remain static for the duration of a PTE scan.
2012      * faults_cpu: Track the nodes the process was running on when a NUMA
2013      * hinting fault was incurred.
2014      * faults_memory_buffer and faults_cpu_buffer: Record faults per node
2015      * during the current scan window. When the scan completes, the counts
2016      * in faults_memory and faults_cpu decay and these values are copied.
2017      */
2018     unsigned long *numa_faults;
2019     unsigned long total_numa_faults;
2020 
2021     /*
2022      * numa_faults_locality tracks if faults recorded during the last
2023      * scan window were remote/local or failed to migrate. The task scan
2024      * period is adapted based on the locality of the faults with different
2025      * weights depending on whether they were shared or private faults
2026      */
2027     unsigned long numa_faults_locality[3];
2028 
2029     unsigned long numa_pages_migrated;
2030 #endif /* CONFIG_NUMA_BALANCING */
2031 
2032 #ifdef CONFIG_ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH
2033     struct tlbflush_unmap_batch tlb_ubc;
2034 #endif
2035 
2036     struct rcu_head rcu;
2037 
2038     /*
2039      * cache last used pipe for splice
2040      */
2041     struct pipe_inode_info *splice_pipe;
2042 
2043     struct page_frag task_frag;
2044 
2045 #ifdef  CONFIG_TASK_DELAY_ACCT
2046     struct task_delay_info *delays;
2047 #endif
2048 #ifdef CONFIG_FAULT_INJECTION
2049     int make_it_fail;
2050 #endif
2051     /*
2052      * when (nr_dirtied >= nr_dirtied_pause), it's time to call
2053      * balance_dirty_pages() for some dirty throttling pause
2054      */
2055     int nr_dirtied;
2056     int nr_dirtied_pause;
2057     unsigned long dirty_paused_when; /* start of a write-and-pause period */
2058 
2059 #ifdef CONFIG_LATENCYTOP
2060     int latency_record_count;
2061     struct latency_record latency_record[LT_SAVECOUNT];
2062 #endif
2063     /*
2064      * time slack values; these are used to round up poll() and
2065      * select() etc timeout values. These are in nanoseconds.
2066      */
2067     u64 timer_slack_ns;
2068     u64 default_timer_slack_ns;
2069 
2070 #ifdef CONFIG_KASAN
2071     unsigned int kasan_depth;
2072 #endif
2073 #ifdef CONFIG_FUNCTION_GRAPH_TRACER
2074     /* Index of current stored address in ret_stack */
2075     int curr_ret_stack;
2076     /* Stack of return addresses for return function tracing */
2077     struct ftrace_ret_stack *ret_stack;
2078     /* time stamp for last schedule */
2079     unsigned long long ftrace_timestamp;
2080     /*
2081      * Number of functions that haven't been traced
2082      * because of depth overrun.
2083      */
2084     atomic_t trace_overrun;
2085     /* Pause for the tracing */
2086     atomic_t tracing_graph_pause;
2087 #endif
2088 #ifdef CONFIG_TRACING
2089     /* state flags for use by tracers */
2090     unsigned long trace;
2091     /* bitmask and counter of trace recursion */
2092     unsigned long trace_recursion;
2093 #endif /* CONFIG_TRACING */
2094 #ifdef CONFIG_KCOV
2095     /* Coverage collection mode enabled for this task (0 if disabled). */
2096     enum kcov_mode kcov_mode;
2097     /* Size of the kcov_area. */
2098     unsigned    kcov_size;
2099     /* Buffer for coverage collection. */
2100     void        *kcov_area;
2101     /* kcov desciptor wired with this task or NULL. */
2102     struct kcov *kcov;
2103 #endif
2104 #ifdef CONFIG_MEMCG
2105     struct mem_cgroup *memcg_in_oom;
2106     gfp_t memcg_oom_gfp_mask;
2107     int memcg_oom_order;
2108 
2109     /* number of pages to reclaim on returning to userland */
2110     unsigned int memcg_nr_pages_over_high;
2111 #endif
2112 #ifdef CONFIG_UPROBES
2113     struct uprobe_task *utask;
2114 #endif
2115 #if defined(CONFIG_BCACHE) || defined(CONFIG_BCACHE_MODULE)
2116     unsigned int    sequential_io;
2117     unsigned int    sequential_io_avg;
2118 #endif
2119 #ifdef CONFIG_DEBUG_ATOMIC_SLEEP
2120     unsigned long   task_state_change;
2121 #endif
2122     int pagefault_disabled;
2123 /* CPU-specific state of this task */
2124     struct thread_struct thread;
2125 /*
2126  * WARNING: on x86, 'thread_struct' contains a variable-sized
2127  * structure.  It *MUST* be at the end of 'task_struct'.
2128  *
2129  * Do not put anything below here!
2130  */
2131 };
```

