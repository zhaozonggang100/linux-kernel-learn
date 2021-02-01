**ref**  

http://blog.chinaunix.net/uid-69947851-id-5825901.html

---

### 1、info

### 2、source

```c++
// file: kernel/exit.c
SYSCALL_DEFINE1(exit, int, error_code)
{
        // 传入进程退出时的退出码，0是正常，其他是异常
        do_exit((error_code&0xff)<<8);
}

// file：kernel/exit.c
// 下面是exit系统调用的核心
void do_exit(long code)
{    
        struct task_struct *tsk = current;
        int group_dead;
        TASKS_RCU(int tasks_rcu_i);
     
        // 处理exit通知链
        profile_task_exit(tsk);
        kcov_task_exit(tsk);
     
        WARN_ON(blk_needs_flush_plug(tsk));
     
        // 中断不能exit
        if (unlikely(in_interrupt()))
                panic("Aiee, killing interrupt handler!");
        // idle进程的pid为0，idle进程不能exit
        if (unlikely(!tsk->pid))
                panic("Attempted to kill the idle task!");
        
        /*
         * If do_exit is called because this processes oopsed, it's possible
         * that get_fs() was left as KERNEL_DS, so reset it to USER_DS before
         * continuing. Amongst other possible reasons, this is to prevent
         * mm_release()->clear_child_tid() from writing to a user-controlled
         * kernel address.
         * 设定进程可以使用的虚拟地址的上限（用户空间）
         * http://lxr.free-electrons.com/ident?v=4.6;i=set_fs
         */
        set_fs(USER_DS);

        ptrace_event(PTRACE_EVENT_EXIT, code);

        validate_creds_for_do_exit(tsk);

        /*
         * We're taking recursive faults here in do_exit. Safest is to just
         * leave this task alone and wait for reboot.
         * 首先是检查PF_EXITING标识, 此标识表示进程正在退出
         */
        if (unlikely(tsk->flags & PF_EXITING)) {
                pr_alert("Fixing recursive fault but reboot is needed!\n");
                /*
                 * We can do this unlocked here. The futex code uses
                 * this flag just to verify whether the pi state
                 * cleanup has been done or not. In the worst case it
                 * loops once more. We pretend that the cleanup was
                 * done as there is no way to return. Either the
                 * OWNER_DIED bit is set by now or we push the blocked
                 * task into the wait for ever nirwana as well.
                 */
                // 如果PF_EXITING已被设置, 则进一步设置PF_EXITPIDONE标识
                tsk->flags |= PF_EXITPIDONE;
                /* 设置进程状态为不可中断的等待状态 */
                set_current_state(TASK_UNINTERRUPTIBLE);
                /* 调度其它进程 */
                schedule();
        }

        // 如果上面检测到进程的PF_EXITING没有置位，这里设置PF_EXITING位
        exit_signals(tsk);  /* sets PF_EXITING */

        sched_exit(tsk);
        schedtune_exit_task(tsk);

        /*
         * tsk->flags are checked in the futex code to protect against
         * an exiting task cleaning up the robust pi futexes.
         */
        // 内存屏障，用于确保在它之后的操作开始执行之前，它之前的操作已经完成
        smp_mb();
        // 一直等待，直到获得current->pi_lock自旋锁
        raw_spin_unlock_wait(&tsk->pi_lock);

        if (unlikely(in_atomic())) {
                pr_info("note: %s[%d] exited with preempt_count %d\n",
                        current->comm, task_pid_nr(current),
                        preempt_count());
                preempt_count_set(PREEMPT_ENABLED);
        }

        /* sync mm's RSS info before statistics gathering */
        // 同步进程的mm的rss_stat
        if (tsk->mm)
                sync_mm_rss(tsk->mm);
        // 更新进程的运行时间, 获取current->mm->rss_stat.count[member]计数 
        // http://lxr.free-electrons.com/source/kernel/tsacct.c?v=4.6#L152
        acct_update_integrals(tsk);
        group_dead = atomic_dec_and_test(&tsk->signal->live);
        if (group_dead) {
                hrtimer_cancel(&tsk->signal->real_timer);
                exit_itimers(tsk->signal);
                if (tsk->mm)
                        setmax_mm_hiwater_rss(&tsk->signal->maxrss, tsk->mm);
        }
        // 收集进程会计信息
        acct_collect(code, group_dead);
        // 记录审计事件
        if (group_dead)
                tty_audit_exit();
        // 释放struct audit_context结构体
        audit_free(tsk);

        tsk->exit_code = code;
        taskstats_exit(tsk, group_dead);

        // 释放存储空间，放弃进程占用的mm,如果没有其他进程使用该mm，则释放它
        exit_mm(tsk);

        // 输出进程会计信息
        if (group_dead)
                acct_process();
        trace_sched_process_exit(tsk);

        // 释放用户空间的“信号量”
        exit_sem(tsk);
        // 释放锁
        exit_shm(tsk);
        // 释放文件对象相关资源
        exit_files(tsk);
        exit_fs(tsk);
        // 脱离控制终端
        if (group_dead)
                disassociate_ctty(1);
        // 释放命名空间
        exit_task_namespaces(tsk);
        exit_task_work(tsk);
        // 释放task_struct中的thread_struct结构
        exit_thread(tsk);

        /*
         * Flush inherited counters to the parent - before the parent
         * gets woken up by child-exit notifications.
         *
         * because of cgroup mode, must be called before cgroup_exit()
         */
        // Performance Event功能相关资源的释放
        perf_event_exit_task(tsk);
        // Performance Event功能相关资源的释放
        cgroup_exit(tsk);

        /*
         * FIXME: do that only when needed, using sched_exit tracepoint
         */
        // 注销断点
        flush_ptrace_hw_breakpoint(tsk);

        TASKS_RCU(preempt_disable());
        TASKS_RCU(tasks_rcu_i = __srcu_read_lock(&tasks_rcu_exit_srcu));
        TASKS_RCU(preempt_enable());
        // 重点***，收完死者值钱的东西后，通知亲属，处理后事，下面单独详解该函数
        exit_notify(tsk, group_dead);
        // 进程事件连接器（通过它来报告进程fork、exec、exit以及进程用户ID与组ID的变化）
        proc_exit_connector(tsk);
#ifdef CONFIG_NUMA
        // 用于NUMA，当引用计数为0时，释放struct mempolicy结构体所占用的内存
        task_lock(tsk);
        mpol_put(tsk->mempolicy);
        tsk->mempolicy = NULL;
        task_unlock(tsk);
#endif
#ifdef CONFIG_FUTEX
        if (unlikely(current->pi_state_cache))
                kfree(current->pi_state_cache);
#endif
        /*
         * Make sure we are holding no locks:
         */
        debug_check_no_locks_held();
        /*
         * We can do this unlocked here. The futex code uses this flag
         * just to verify whether the pi state cleanup has been done
         * or not. In the worst case it loops once more.
         */
        tsk->flags |= PF_EXITPIDONE;

        if (tsk->io_context)
                exit_io_context(tsk);

        if (tsk->splice_pipe)
                free_pipe_info(tsk->splice_pipe);

        if (tsk->task_frag.page)
                put_page(tsk->task_frag.page);

        validate_creds_for_do_exit(tsk);

        // 检查有多少未使用的进程内核栈
        check_stack_usage();
        preempt_disable();
        if (tsk->nr_dirtied)
                __this_cpu_add(dirty_throttle_leaks, tsk->nr_dirtied);
        exit_rcu();
        TASKS_RCU(__srcu_read_unlock(&tasks_rcu_exit_srcu, tasks_rcu_i));

        /*
         * The setting of TASK_RUNNING by try_to_wake_up() may be delayed
         * when the following two conditions become true.
         *   - There is race condition of mmap_sem (It is acquired by
         *     exit_mm()), and
         *   - SMI occurs before setting TASK_RUNINNG.
         *     (or hypervisor of virtual machine switches to other guest)
         *  As a result, we may become TASK_RUNNING after becoming TASK_DEAD
         *
         * To avoid it, we have to wait for releasing tsk->pi_lock which
         * is held by try_to_wake_up()
         */
        smp_mb();
        raw_spin_unlock_wait(&tsk->pi_lock);

        /* causes final put_task_struct in finish_task_switch(). */
        /*
            在设置了进程状态为TASK_DEAD后, 进程进入僵死状态, 进程已经无法被再次调度, 因为对应用程序或者用户空间来说此进程已经死了, 但是尽管进程已经不能再被调度，但系统还是保留了它的进程描述符，这样做是为了让系统有办法在进程终止后仍能获得它的信息。
            在父进程获得已终止子进程的信息后，子进程的task_struct结构体才被释放（包括此进程的内核栈）。
        */
        tsk->state = TASK_DEAD;
        tsk->flags |= PF_NOFREEZE;      /* tell freezer to ignore us */
        schedule();
        // 下面不应该被执行到？？？
        BUG();
        /* Avoid "noreturn function does return".  */
        for (;;)
                cpu_relax();    /* For when BUG is null */
}
EXPORT_SYMBOL_GPL(do_exit);

/*
 * Send signals to all our closest relatives so that they know
 * to properly mourn us..
 */
// file：kernel/exit.c
static void exit_notify(struct task_struct *tsk, int group_dead)
{
        bool autoreap;
        struct task_struct *p, *n;
        LIST_HEAD(dead); 

        write_lock_irq(&tasklist_lock);
        // *** 下面详细介绍该函数 ***
        // 因该tsk死亡了，这里需要为tsk的子进程选取新的父进程：学名叫刘备托孤，当然没有儿子也就不用托孤了，这函数也会直接返回
        // 把该进程的所有子孙进程重设父进程，交给init进程接管
        forget_original_parent(tsk, &dead);

        // 世事无常，这一脉的线程组里已经空了，那么如果这个死者所在的进程组是孤儿进程组，则向这个孤儿组内还有stop job的线程发送SIGHUP和SIGCONT信号吧
        if (group_dead)
                kill_orphaned_pgrp(tsk->group_leader, NULL);

        if (unlikely(tsk->ptrace)) {
                // 当前进程被trace
                int sig = thread_group_leader(tsk) &&
                                thread_group_empty(tsk) &&
                                !ptrace_reparented(tsk) ?
                        tsk->exit_signal : SIGCHLD;
                autoreap = do_notify_parent(tsk, sig);
        } else if (thread_group_leader(tsk)) {
                //当前线程为线程组组长，如果线程组不为空，就不要通知组长的父进程它已经死了的消息，即autoreap=false
                // 即这个线程组即使为僵尸，也不回收它的资源
                autoreap = thread_group_empty(tsk) &&
                        do_notify_parent(tsk, tsk->exit_signal);
        } else {
                // 如果线程组为空，就通知它的父进程，死亡原因（主要是回收资源相关）
                autoreap = true;
        }

        // 这里如果do_notify_parent返回是autoreap =true即EXIT_DEAD,说明父进程不需要子进程的退出码，即父进程不需要调用wait/waitpid等函数；否则就设置成EXIT_ZOMBIE,等父进程调用了wait函数后，再释放这个进程的task_struct
        tsk->exit_state = autoreap ? EXIT_DEAD : EXIT_ZOMBIE;
        // 如果进程已死，父进程不需要waitpid, 就放到资源释放链表dead里面去，等待释放
        if (tsk->exit_state == EXIT_DEAD)
                list_add(&tsk->ptrace_entry, &dead);

        /* mt-exec, de_thread() is waiting for group leader */
        // 引用计数小于0，唤醒线程组退出task去执行
        if (unlikely(tsk->signal->notify_count < 0))
                wake_up_process(tsk->signal->group_exit_task);
        write_unlock_irq(&tasklist_lock);

        // 遍历资源释放链表dead，为每一个在表里的释放资源，如果释放到当前线程，并且线程已空，而且线程组组长也是僵尸，那么久将线程组组长一起释放了
        list_for_each_entry_safe(p, n, &dead, ptrace_entry) {
                list_del_init(&p->ptrace_entry);
                release_task(p);
        }
}

/*              
 * This does two things:
 *      
 * A.  Make init inherit all the child processes
 * B.  Check to see if any process groups have become orphaned
 *      as a result of our exiting, and if they have any stopped
 *      jobs, send them a SIGHUP and then a SIGCONT.  (POSIX 3.2.2.2)
 */ 
// file：kernel/exit.c 
//从上面可以看出，死者拜托函数forget_original_parent进行托孤(pid空间的1号进程不是指pid=1,而是指它是这个命名空间的第一个进程。通常情况只有一个命名空间，而这个命名空间的1号进程为init进程)           
static void forget_original_parent(struct task_struct *father,
                                        struct list_head *dead)
{         
        struct task_struct *p, *t, *reaper;
        // 当前进程如果为ptrace，则退出所有ptrace下面的进程    
        if (unlikely(!list_empty(&father->ptraced)))
                exit_ptrace(father, dead); // 退出所有被trace的task，并将已死的进程加入到dead链表

        /* Can drop and reacquire tasklist_lock */
        // 选择合适的默认收养者(继父)，通常为当前pid空间的1号进程，我们叫它村长，只有一个ns的情况下就是init进程
        reaper = find_child_reaper(father);
        // 当前进程没有子进程，就直接返回了，不需要托孤了
        if (list_empty(&father->children))
                return;
 
        // 作为该pid空间的默认收养者，虽然有承担收养遗孤的责任，但是也不能什么孤儿都往这里放啊，还是要先看看孤儿的亲戚有没有愿意收养的吧
        // 尝试从亲戚中挑选一个收养者，pid命名空间的1号进程（村长）作为缺省的收养者
        reaper = find_new_reaper(father, reaper);
        // 将当前进程下的所有娃儿的父亲都改成reaper（这个reaper可能是村长，也可能是他的祖先）
        list_for_each_entry(p, &father->children, sibling) {
                for_each_thread(p, t) {
                        // 设置新的父进程real_parent ，在有调试的时候，parent 指向ptrace进程
                        t->real_parent = reaper;
                        BUG_ON((!t->ptrace) != (t->parent == father));
                        if (likely(!t->ptrace))
                                t->parent = t->real_parent;
                        if (t->pdeath_signal)
                                group_send_sig_info(t->pdeath_signal,
                                                    SEND_SIG_NOINFO, t);
                }
                /*
                 * If this is a threaded reparent there is no need to
                 * notify anyone anything has happened.
                 */
                // 如果死者和继父不是同一个线程组（不是一家）
                // (继父这里很可能就是init进程或者Pid空间的1号进程)，那没办法，把死者的所有儿子状态为
                // EXIT_ZOMBIE的告诉它的继父；同时如果变成了孤儿组，则向孤儿组内所有还有stop jobs的进程
                // 发送SIGHUP和SIGCONT信号。
                if (!same_thread_group(reaper, father))
                        reparent_leader(father, p, dead);
        }
        // 将当前进程的子进程 全部加到继父进程的链表上，从此这个继父就是这些子进程的父亲
        list_splice_tail_init(&father->children, &reaper->children);
}

// file：kernel/exit.c
static struct task_struct *find_child_reaper(struct task_struct *father)
        __releases(&tasklist_lock)
        __acquires(&tasklist_lock)
{
        struct pid_namespace *pid_ns = task_active_pid_ns(father);
        struct task_struct *reaper = pid_ns->child_reaper;

        // 查看死者是否为所在的pid空间的1号进程（村长） 如果不是，就返回pid空间的1号进程，作为缺省收养者
        if (likely(reaper != father))
                return reaper;

        // pid空间的1号进程就是死者，则需要从死者所在的线程组选择一个亲戚作为备用收养者（下一个活动线程）
        // 哎，程序也讲究 世袭制，村长从亲戚中选
        reaper = find_alive_thread(father);
        if (reaper) {
                // 设置这个线程为当前pid空间的1号进程（村长）
                pid_ns->child_reaper = reaper;
                return reaper;
        }

        write_unlock_irq(&tasklist_lock);
        // 如果当前进程没有线程，并且自己又是1号进程，而且还是init_pid_ns,说明退出的是init进程，抛出异常
        // 创世神已死，留着世界干啥，panic掉
        if (unlikely(pid_ns == &init_pid_ns)) {
                panic("Attempted to kill init! exitcode=0x%08x\n",
                        father->signal->group_exit_code ?: father->exit_code);
        }
        zap_pid_ns_processes(pid_ns);
        write_lock_irq(&tasklist_lock);

        // 死者继续上，没办法，不安宁
        return father;
}

/*
 * When we die, we re-parent all our children, and try to:
 * 1. give them to another thread in our thread group, if such a member exists
 * 2. give it to the first ancestor process which prctl'd itself as a
 *    child_subreaper for its children (like a service manager)
 * 3. give it to the init process (PID 1) in our pid namespace
 */      
// 由于村长不同意总是收养孤儿，因此，拜托find_new_reaper函数去找一个更合适的收养者，如果实在没有找到，再找他吧
// file：kernel/exit.c
static struct task_struct *find_new_reaper(struct task_struct *father,
                                           struct task_struct *child_reaper)
{
        struct task_struct *thread, *reaper;

        // child_reaper为find_child_reaper获取到的推荐收养者，但是收养者不同意，要求先从它的兄弟姐妹选择
        // 从当前进程的线程组（兄弟姐妹）选择一个alive的线程，如果线程存在，就用这个线程作为收养者
        thread = find_alive_thread(father); 
        if (thread)     
                return thread;  

        // 这货没有兄弟姐妹，就只有找找他是否有遗属，说明他祖宗愿不愿意作为收养者，如果有，就往上去找他的祖宗来收养， has_child_subreaper为真，表示死者有交代他们家有祖先愿意做收养者             
        if (father->signal->has_child_subreaper) {
                //从死者往上找，直到child_reaper为止，如果找到备胎愿意收养，就用这个祖宗线程组下
                //的线程作为收养者。如果找到init了，也没找到，那就只要让pid空间的1号来收养了，能者多劳嘛。
                /*                                  
                 * Find the first ->is_child_subreaper ancestor in our pid_ns.
                 * We start from father to ensure we can not look into another
                 * namespace, this is safe because all its threads are dead.
                 */
                for (reaper = father;
                     !same_thread_group(reaper, child_reaper);
                     reaper = reaper->real_parent) {
                        /* call_usermodehelper() descendants need this check */
                        /* 都找到创世神这里了，都没人愿意收养，只有跳出去找对应pid空间的1号进程了*/
                        if (reaper == &init_task)
                                break;
                        // 这个祖先表明，他就是那个收养者，如果不是就continue
                        if (!reaper->signal->is_child_subreaper)
                                continue;
                        // 找到一个合适的祖先，从这个辈分的祖先里面选一个或者的老者来收养
                        thread = find_alive_thread(reaper);
                        if (thread)
                                return thread;
                }
        }

        // 村长：我曹，果然还是我,不让人省心啊
        return child_reaper;
}

/*
* Any that need to be release_task'd are put on the @dead list.
 */
// file：kernel/exit.c
static void reparent_leader(struct task_struct *father, struct task_struct *p,
                                struct list_head *dead)
{
        // 进程已经死了，就直接返回了
        if (unlikely(p->exit_state == EXIT_DEAD))
                return;

        /* We don't want people slaying init. */
        p->exit_signal = SIGCHLD;

        /* If it has exited notify the new parent about this child's death. */
        // 查看进程是否为僵尸，并且所在线程组为空，就向它的新父亲，发送SIGCHLD信号
        if (!p->ptrace &&
            p->exit_state == EXIT_ZOMBIE && thread_group_empty(p)) {
                if (do_notify_parent(p, p->exit_signal)) {
                        p->exit_state = EXIT_DEAD;
                        list_add(&p->ptrace_entry, dead);
                }
        }

        // 如果是孤儿进组，则向进程组内所有还有stop jobs的进程发送SIGHUP和SIGCONT信号。
        kill_orphaned_pgrp(p, father);
}

/*      
 * Check to see if any process groups have become orphaned as
 * a result of our exiting, and if they have any stopped jobs,
 * send them a SIGHUP and then a SIGCONT. (POSIX 3.2.2.2)
 */
// UNIX中 在父进程终止后，进程组成为孤儿进程组，POSIX要求向新的孤儿进程组中处于停止状态的每一个进程发送挂断信号（SIGHUP），接着又向其发送继续信号（SIGCONT）
// file：kernel/exit.c
static void
kill_orphaned_pgrp(struct task_struct *tsk, struct task_struct *parent)
{
        struct pid *pgrp = task_pgrp(tsk);
        struct task_struct *ignored_task = tsk;

        if (!parent)
                /* exit: our father is in a different pgrp than
                 * we are and we were the only connection outside.
                 */
                parent = tsk->real_parent;
        else
                /* reparent: our child is in a different pgrp than
                 * we are, and it was the only connection outside.
                 */
                ignored_task = NULL;

        // UNIX中 在父进程终止后，进程组成为孤儿进程组，POSIX要求向新的孤儿进程组中处于停止状态的
        // 每一个进程发送挂断信号（SIGHUP），接着又向其发送继续信号（SIGCONT）
        if (task_pgrp(parent) != pgrp &&
            task_session(parent) == task_session(tsk) && 
            will_become_orphaned_pgrp(pgrp, ignored_task) &&
            has_stopped_jobs(pgrp)) {
                __kill_pgrp_info(SIGHUP, SEND_SIG_PRIV, pgrp);
                __kill_pgrp_info(SIGCONT, SEND_SIG_PRIV, pgrp);
        }       
}

/*
    release_task释放task_struct资源的函数
*/
// file：kernel/exit.c
void release_task(struct task_struct *p)
{
        struct task_struct *leader;
        int zap_leader;
repeat:
        /* don't need to get the RCU readlock here - the process is dead and
         * can't be modifying its own credentials. But shut RCU-lockdep up */
        rcu_read_lock();
        atomic_dec(&__task_cred(p)->user->processes);
        rcu_read_unlock();

        proc_flush_task(p);

        write_lock_irq(&tasklist_lock);
        ptrace_release_task(p);
        __exit_signal(p);

        /*
         * If we are the last non-leader member of the thread
         * group, and the leader is zombie, then notify the
         * group leader's parent process. (if it wants notification.)
         */ 
        // 查看线程组是否已经没有线程存活了，如果已经空了，就通知线程组组长的父进程对组长进行收尸
        zap_leader = 0;
        leader = p->group_leader;
        if (leader != p && thread_group_empty(leader)
                        && leader->exit_state == EXIT_ZOMBIE) {
                /*
                 * If we were the last child thread and the leader has
                 * exited already, and the leader's parent ignores SIGCHLD,
                 * then we are the one who should release the leader.
                 */
                zap_leader = do_notify_parent(leader, leader->exit_signal);
                if (zap_leader)
                        leader->exit_state = EXIT_DEAD;
        }

        write_unlock_irq(&tasklist_lock);
        release_thread(p);
        //释放线程资源,注意当前线程的task_struct在这里是不会释放的（引用计数不为0），因为目前还使用的它的进程上下文，schedule还需要他参与，它会在switch_to函数执行之后释放。
        call_rcu(&p->rcu, delayed_put_task_struct);

        p = leader;
        if (unlikely(zap_leader))
                goto repeat;
}

/*
 * Let a parent know about the death of a child.
 * For a stopped/continued status change, use do_notify_parent_cldstop instead.
 *                      
 * Returns true if our parent ignored us and so we've switched to
 * self-reaping.
 */
// kernel/signal.c             
bool do_notify_parent(struct task_struct *tsk, int sig) 
{                               
        struct siginfo info;
        unsigned long flags;
        struct sighand_struct *psig;
        bool autoreap = false;
        cputime_t utime, stime;

        BUG_ON(sig == -1);

        /* do_notify_parent_cldstop should have been called instead.  */
        BUG_ON(task_is_stopped_or_traced(tsk));
                
        BUG_ON(!tsk->ptrace &&
               (tsk->group_leader != tsk || !thread_group_empty(tsk)));
 
        if (sig != SIGCHLD) {
                /*
                 * This is only possible if parent == real_parent.
                 * Check if it has changed security domain.
                 */
                if (tsk->parent_exec_id != tsk->parent->self_exec_id)
                        sig = SIGCHLD;
        }

        // 初始化信号发送结构SIGCHLD
        info.si_signo = sig;
        info.si_errno = 0;
        /*
         * We are under tasklist_lock here so our parent is tied to
         * us and cannot change.
         *
         * task_active_pid_ns will always return the same pid namespace
         * until a task passes through release_task.
         *
         * write_lock() currently calls preempt_disable() which is the
         * same as rcu_read_lock(), but according to Oleg, this is not
         * correct to rely on this
         */
        rcu_read_lock();
        info.si_pid = task_pid_nr_ns(tsk, task_active_pid_ns(tsk->parent));
        info.si_uid = from_kuid_munged(task_cred_xxx(tsk->parent, user_ns),
                                       task_uid(tsk));
        rcu_read_unlock();

        task_cputime(tsk, &utime, &stime);
        info.si_utime = cputime_to_clock_t(utime + tsk->signal->utime);
        info.si_stime = cputime_to_clock_t(stime + tsk->signal->stime);

        info.si_status = tsk->exit_code & 0x7f;
        if (tsk->exit_code & 0x80)
                info.si_code = CLD_DUMPED;
        else if (tsk->exit_code & 0x7f)
                info.si_code = CLD_KILLED;
        else {
                info.si_code = CLD_EXITED;
                info.si_status = tsk->exit_code >> 8;
        }

        psig = tsk->parent->sighand;
        spin_lock_irqsave(&psig->siglock, flags);
        if (!tsk->ptrace && sig == SIGCHLD &&
            (psig->action[SIGCHLD-1].sa.sa_handler == SIG_IGN ||
             (psig->action[SIGCHLD-1].sa.sa_flags & SA_NOCLDWAIT))) {
                // 父进程说：它要为儿子收尸，所以autoreap = true;后面会设置为僵尸进程，如果父进程一直不收，则一直处理僵尸状态 
                /*
                 * We are exiting and our parent doesn't care.  POSIX.1
                 * defines special semantics for setting SIGCHLD to SIG_IGN
                 * or setting the SA_NOCLDWAIT flag: we should be reaped
                 * automatically and not left for our parent's wait4 call.
                 * Rather than having the parent do it as a magic kind of
                 * signal handler, we just set this to tell do_exit that we
                 * can be cleaned up without becoming a zombie.  Note that
                 * we still call __wake_up_parent in this case, because a
                 * blocked sys_wait4 might now return -ECHILD.
                 *
                 * Whether we send SIGCHLD or not for SA_NOCLDWAIT
                 * is implementation-defined: we do (if you don't want
                 * it, just use SIG_IGN instead).
                 */
                autoreap = true;
                if (psig->action[SIGCHLD-1].sa.sa_handler == SIG_IGN)
                        sig = 0;
        }
        // 给父进程发一个SIGCHLD信号，通知收尸
        if (valid_signal(sig) && sig)
            __group_send_sig_info(sig, &info, tsk->parent);
        // 唤醒父进程
        __wake_up_parent(tsk, tsk->parent);
        spin_unlock_irqrestore(&psig->siglock, flags);

        return autoreap;
}
```

### 3、panic实例

**ref** : `https://winddoing.github.io/post/7653.html`

**出错日志** : `Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000000b`

**函数调用** ：  
&emsp;&emsp; do_exit->exit_notify->forget_original_parent(tsk)->find_new_reaper(father)->`"Attempted to kill init! exitcode=0x%08x\n"`

**详解**：  

&emsp;&emsp; 1、`task_struct->exit_code`是32位的整数，高24位保留，低8位用于表示进程的退出状态（**高4位代表退出码，低4位代表进程退出的信号标识**）进程退出时调用`do_exit((error_code&0xff)<<8)`，只有低8位有效

&emsp;&emsp; 2、上面的出错日志表明进程退出时的信号标识为11，查看信号表得知是init进程发生了**段错误**
```c++
#define SIGHUP           1
#define SIGINT           2
#define SIGQUIT          3
#define SIGILL           4
#define SIGTRAP          5
#define SIGABRT          6
#define SIGIOT           6
#define SIGBUS           7
#define SIGFPE           8
#define SIGKILL          9
#define SIGUSR1         10
#define SIGSEGV         11
#define SIGUSR2         12
#define SIGPIPE         13
#define SIGALRM         14
#define SIGTERM         15
#define SIGSTKFLT       16
#define SIGCHLD         17
#define SIGCONT         18
#define SIGSTOP         19
#define SIGTSTP         20
#define SIGTTIN         21
#define SIGTTOU         22
#define SIGURG          23
#define SIGXCPU         24
#define SIGXFSZ         25
#define SIGVTALRM       26 
#define SIGPROF         27
#define SIGWINCH        28
#define SIGIO           29
#define SIGPOLL         SIGIO
/*
#define SIGLOST         29
*/
#define SIGPWR          30
#define SIGSYS          31
#define SIGUNUSED       31
/* These should not be considered constants from userland.  */
#define SIGRTMIN        32
#ifndef SIGRTMAX
#define SIGRTMAX        _NSIG
#endif
```