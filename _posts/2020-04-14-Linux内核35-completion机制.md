---
layout:     post
title:      Linux内核35-Completion机制
subtitle:   Linux同步之Completion机制的工作原理以及实现
date:       2020-04-14
author:     Tupelo Shen
header-img: img/post-bg-unix-linux.jpg
catalog:    true
tags:
    - Linux
    - Linux内核
    - Completion
---

每一种技术的出现必然是因为某种需求。正因为人的本性是贪婪的，所以科技的创新才能日新月异。

# Completion机制的工作原理

内核编程中的一个常见模式就是在当前进程中，再去启动另外一个活动，比如创建新的内核线程或用户进程、向已存在的进程发起请求、再或者操作某些硬件。针对这些情况，内核当然可以尝试使用信号量同步两个任务，代码如下所示：

    struct semaphore sem;

    init_MUTEX_LOCKED(&sem);
    start_external_task(&sem);
    down(&sem);

把信号量初始化为一个关闭的互斥信号量，也就是count=0，然后启动外部任务并挂起等待信号量的释放。当外部的任务完成操作后，调用up(&sem)释放信号量，上面的代码继续往下执行。

正常逻辑下，上面的代码一点毛病没有。但世上的事就没有完美的。我们假设两种异常情况：第一种情况是，如果上面的代码是一个通信任务的话（我们都知道，通信任务一般对信号量的竞争都比较激烈），性能往往会变得非常糟糕，因为调用down()函数的进程几乎总是处于等待之中。第二种情况是，在多核系统中，假设定义的信号量只是一个临时变量，按照上面的调用关系，上面的代码一旦被唤醒就要销毁临时信号量的话，这个进程启动的外部任务很可能还处于执行up()函数的过程中。而此时，信号量已经被销毁，up()函数可能会尝试访问一个不存在的信号量数据结构。当然了，第二种情况可以使用其它指令，禁止down()和up()函数的并发执行。但是，这样的话，又增加了新的负荷。所以，并不是一个特别好的选择。

针对上面的情况，Linux内核从2.4.7版本开始，引入了另外一种同步技术：completion机制。

# Completion机制的数据结构

completion同步原语的数据结构如下代码所示：

    struct completion {
        unsigned int done;
        wait_queue_head_t wait;
    };

可以看出，其由一个整形数done和队列head组成。

# Completion机制的常用API

与信号量的up()函数对应的函数称为complete()函数。它的参数是一个completion数据结构。这个函数会调用spin_lock_irqsave()函数，请求completion等待队列的保护自旋锁，增加done的值，唤醒等待队列中的休眠进程中的一个，最后调用spin_unlock_irqrestore()释放自旋锁。

与信号量的down()函数对应的称为wait_for_completion()函数。它的参数也是completion数据结构。这个函数会检查done的值：如果大于0，函数执行终止，因为另一个CPU上已经执行了complete()函数；否则，这个函数添加当前进程到等待队列的队尾，并使进程进入休眠，将其进程状态设为TASK_UNINTERRUPTIBLE（如果代码调用了该函数，而且被等待的任务没有完成，结果就是，等待的任务就是一个不可杀的进程）。一旦进程被唤醒，这个函数就会把当前进程从等待队列中删除。然后，再次检查done的值，如果等于0，则函数执行终止；否则，当前进程会再次被挂起。同complete()函数一样，这个函数也使用自旋锁保护等待队列。

completion和信号量的真正区别是等待队列中的自旋锁如何使用。在completion中，自旋锁被用来保证complete()和wait_for_completion()不会并发执行。在信号量中，自旋锁被用来保证并发执行的两个调用down()的函数不会弄乱信号量数据结构。

# Completion机制的示例

关于completion机制如何使用，请参考complete的模块示例。该模块定义了一个这样的模块：任何尝试读取设备的进程都会进入等待状态（通过调用wait_for_completion()函数实现），直到有其它进行尝试写该设备。代码类似于下面的代码：

    DECLARE_COMPLETION(comp);
    ssize_t complete_read (struct file *filp, char __user *buf, size_t count, loff_t
            *pos)
    {
        printk(KERN_DEBUG "process %i (%s) going to sleep\n",
                current->pid, current->comm);
        wait_for_completion(&comp);
        printk(KERN_DEBUG "awoken %i (%s)\n", current->pid, current->comm);
        return 0;
    }
    ssize_t complete_write (struct file *filp, const char __user *buf, size_t count,
            loff_t *pos)
    {
        printk(KERN_DEBUG "process %i (%s) awakening the readers...\n",
                current->pid, current->comm);
        complete(&comp);
        return count; /* 成功，避免重试 */
    }

在上面的示例中，可能存在多个进程同时读取设备。对设备的一次写操作只能试一个读操作完成，而无法通知其它正在读操作的进程。

completion机制的一个典型应用就是，在模块exit的时候，终止内核线程。在一些典型的例子中，驱动程序的内部工作是在内核线程中使用while(1)循环中实现的。当模块准备好清理时，exit函数就会告诉线程需要退出，然后等待线程的completion事件。基于这个目的，内核提供了一个特殊的函数供线程调用：

    void complete_and_exit(struct completion *c, long retval);