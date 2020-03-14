---
layout:     post
title:      Linux内核15-_do_fork()
subtitle:   分析Linux内核创建进程的过程
date:       2020-03-14
author:     Tupelo Shen
header-img: img/post-bg-unix-linux.jpg
catalog:    true
tags:
    - Linux
    - Linux内核
    - clone
    - fork
    - vfork
---
# 1. `_do_fork()`函数

不论是clone()、fork()还是vfork()，它们最核心的部分还是调用_do_fork()（一个与体系无关的函数），完成创建进程的工作。它具有如下参数：

> 早期版本中是调用`do_fork()`函数。其实，`_do_fork`和`do_fork`在进程的复制的时候并没有太大的区别, 他们就只是在进程tls复制的过程中实现有细微差别

下面是_do_fork的源代码:

    long _do_fork(unsigned long clone_flags,
              unsigned long stack_start,
              unsigned long stack_size,
              int __user *parent_tidptr,
              int __user *child_tidptr,
              unsigned long tls)
    {
        struct task_struct *p;
        int trace = 0;
        long nr;

        /*
         * 当从kernel_thread调用或CLONE_UNTRACED被设置时，不需要报告event
         * 否则，报告使用哪种fork类型的event
         */
        if (!(clone_flags & CLONE_UNTRACED)) {
            if (clone_flags & CLONE_VFORK)
                trace = PTRACE_EVENT_VFORK;
            else if ((clone_flags & CSIGNAL) != SIGCHLD)
                trace = PTRACE_EVENT_CLONE;
            else
                trace = PTRACE_EVENT_FORK;

            if (likely(!ptrace_event_enabled(current, trace)))
                trace = 0;
        }
        /* 拷贝进程描述符 */
        p = copy_process(clone_flags, stack_start, stack_size,
                 child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
        /*
         * 在唤醒新线程之前，先检查指针是否合法
         * 因为如果线程创建后立即退出的话，线程指针可能会非法
         */
        if (!IS_ERR(p)) {
            struct completion vfork;
            struct pid *pid;

            trace_sched_process_fork(current, p);

            /* 分配PID */
            pid = get_task_pid(p, PIDTYPE_PID);
            nr = pid_vnr(pid);

            /* 把新进程PID写入到父进程的用户态变量中。 */
            if (clone_flags & CLONE_PARENT_SETTID)
                put_user(nr, parent_tidptr);

            /* 如果实现的是vfork调用，则完成completion机制，确保父进程后续运行 */
            if (clone_flags & CLONE_VFORK) {
                p->vfork_done = &vfork;
                init_completion(&vfork);
                get_task_struct(p);
            }

            /* 唤醒新进程 */
            wake_up_new_task(p);

            /* fork过程完成，子进程开始运行，并告知ptracer */
            if (unlikely(trace))
                ptrace_event_pid(trace, pid);

            /*
             * 如果实现的vfork调用，则将父进程插入等待队列，并挂起父进程
             * 直到子进程释放自己的内存空间，也就是，保证子进程先行于父进程
             */
            if (clone_flags & CLONE_VFORK) {
                if (!wait_for_vfork_done(p, &vfork))
                    ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
            }
            put_pid(pid);
        } else {
            nr = PTR_ERR(p);
        }
        return nr;
    }

# 2. copy_process()函数

copy_process函数实现进程创建的大部分工作：创建旧进程的副本，比如进程描述符和子进程运行需要的其它内核数据结构。下面是一段伪代码：

    static struct task_struct *copy_process(unsigned long clone_flags,
                        unsigned long stack_start,
                        unsigned long stack_size,
                        int __user *child_tidptr,
                        struct pid *pid,
                        int trace,
                        unsigned long tls,
                        int node)
    {
        // 1. 检查参数clone_flags，一些标志组合是否合理
        // ...

        // 2. 其它的安全检查
        retval = security_task_create(clone_flags);

        // 3. 分配一个新的task_struct结构，与父进程完全相同，只是stack不同
        p = dup_task_struct(current, node);

        // 4. 检查该用户的进程数是否超过限制
        if (atomic_read(&p->real_cred->user->processes) >=
                task_rlimit(p, RLIMIT_NPROC)) {
            // 检查该用户是否具有相关权限，不一定是root
            if (p->real_cred->user != INIT_USER &&
                // ...
        }

        // 5. 检查进程数量是否超过max_threads，后者取决于内存的大小
        if (nr_threads >= max_threads)
        // ...

        // 6. 进程描述符相关变量初始化

        // 7. 完成调度器相关数据结构的初始化
        retval = sched_fork(clone_flags, p);

        // 8. 拷贝所有的进程信息
        shm_init_task(p);
        retval = copy_semundo(clone_flags, p);
        retval = copy_files(clone_flags, p);
        retval = copy_fs(clone_flags, p);
        retval = copy_sighand(clone_flags, p);
        retval = copy_signal(clone_flags, p);
        retval = copy_mm(clone_flags, p);
        retval = copy_namespaces(clone_flags, p);
        retval = copy_io(clone_flags, p);

        // 9. 设置子进程的pid
        retval = copy_thread_tls(clone_flags, stack_start, stack_size, p, tls);

        // ...

        // 10. 设置子进程的PID
        p->pid = pid_nr(pid);

        // 11. 根据是创建线程还是进程设置线程组组长、进程组组长等等信息
        // ...

        // 12. 将pid加入PIDTYPE_PID这个散列表
        if (likely(p->pid)) {
            // ..
            attach_pid(p, PIDTYPE_PID);
            nr_threads++;
        }

        // 释放资源，善后处理
        return p;
    err:
        // 错误处理
    }

现在，我们已经有了一个可运行的子进程，但是，实际上还没有运行。至于何时运行取决于调度器。在未来的某个进程切换时间点上，调度器把子进程描述符中的thread成员中的值加载到CPU上，赋予子进程CPU的使用权。esp寄存器加载thread.esp的值（也就是获取了子进程的内核态栈的地址），eip寄存器加载ret_from_fork()函数的返回地址（子进程执行的下一条指令）。ret_from_fork()是一个汇编函数，它调用schedule_tail()，继而调用finish_task_switch()函数，完成进程切换，然后把栈上的值加载到寄存器中，强迫CPU进入用户模式。基本上，新进程的执行恰恰在fork()、vfork()或clone()系统调用结束之时。该系统调用的返回值保存在寄存器eax中：对于子进程是0，对于父进程来说就是子进程的PID。

要想理解这一点，应该要重点理解一下copy_thread_tls()函数，其与早期的copy_thread()函数非常类似，只是在末尾添加了向子线程添加tls的内容。

在copy_thread_tls()函数中，我们可以看到这样的代码：

    childregs->ax = 0;
    p->thread.ip = (unsigned long) ret_from_fork;

这就是为什么子进程为什么返回PID=0的原因。

# 总结

1. 这个函数看似很复杂，其实就是根据用户态传递过来的控制参数，实现进程的4要素：执行代码、私有堆栈空间、进程控制块（task_struct）和独立的内存空间。当然了，线程只是拷贝父进程的即可。
2. 创建完进程的4要素之后，把新进程的最开始执行的指令设置到eip寄存器即可。然后就是等待内核调度。当轮到新进程使用CPU的时候，就从eip寄存器开始执行。
