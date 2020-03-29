---
layout:     post
title:      Linux内核22-软中断和tasklet
subtitle:   Linux内核是如何实现和处理软中断和tasklet
date:       2020-03-29
author:     Tupelo Shen
header-img: img/post-bg-unix-linux.jpg
catalog:    true
tags:
    - Linux
    - Linux内核
    - 软中断
    - tasklet
---

# 1 软中断和Tasklet介绍

在之前的文章中，讲解中断处理相关的概念的时候，提到过有些任务不是紧急的，可以延后一段时间执行。因为中断服务例程都是顺序执行的，在响应一个中断的时候不应该被打断。相反，这些可延时任务执行时，可以使能中断。那么，将这些任务从中断处理程序中剥离出来，可以有效地保证内核对于中断响应时间尽可能短。这对于时间苛刻的应用来说，这是一个很重要的属性，尤其是那些要求中断请求必须在毫秒级别响应的应用。

Linux2.6内核使用两种手段满足这项挑战：软中断和tasklet，还有工作队列。其中，工作队列我们单独在一篇文章中讲解。

软中断和tasklet这两个术语是息息相关的，因为tasklet是基于软中断实现的。事实上，出现在内核源代码中的`软中断`概念有时候指的就是这两个术语的统称。另一个广泛使用的术语是`中断上下文`：可以是内核正在执行的中断处理程序，也可以是一个可延时处理的函数。

软中断是静态分配好的（编译时），而tasklet是在运行时分配并初始化的（比如，加载内核模块的时候）。因为软中断的实现是可重入的，使用自旋锁保护它们的数据结构。所以软中断可以在多个CPU上并发运行。tasklet不需要考虑这些，因为它的处理完全由内核控制，也就是说，相同类型的tasklet总是顺序执行的。换句话说，不可能同时有2个以上的CPU执行相同类型的tasklet。当然了，不同类型的tasklet完全可以在多个CPU上同时执行。完全顺序执行的tasklet简化了驱动开发者的工作，因为tasklet不需要考虑可重入设计。

既然已经理解了软中断和tasklet的机制，那么实现这样的可延时函数需要哪些步骤呢？如下所示：

1. 初始化

    定义一个可延时函数。这一步，一般在内核初始化自身或者加载内核模块时完成。

2. 激活

    将上面定义的函数挂起。也就是等待内核下一次的调度执行。激活可以发生在任何时候。

3. 禁止

    对于定义的函数，可以选择性的禁止执行。

4. 执行

    执行定义的延时函数。对于执行的时机，通过软中断控制。

激活和执行是绑定在一起的，也就是说，那个CPU激活延时函数就在那个CPU上执行。但这并不是总能提高系统性能。虽然从理论上说，绑定可延时函数到激活它的CPU上更有利于利用CPU硬件Cache。毕竟，可以想象的是，正在执行的内核线程要访问的数据结构也可能是可延时函数使用的数据。但是，因为等到延时函数执行的时候，已经过了一段时间，Cache中的相关行可能已经不存在了。更重要的是，总是把一个函数绑定到某个CPU上执行是有风险的，这个CPU可能负荷很重而其它的CPU可能比较空闲。

# 2 软中断

Linux2.6内核中，软中断的数量比较少。对于多数目的，这些tasklet足够了。因为不需要考虑重入，所以简单易用。事实上，只使用了6类软中断，如下表所示：

表4-9 Linux2.6中使用的软中断

| 软中断 | 优先级 | 描述 |
| ------ | ------ | ----- |
| HI_SOFTIRQ    | 0 | 处理高优先级的tasklet |
| TIMER_SOFTIRQ | 1 | 定时器中断 |
| NET_TX_SOFTIRQ| 2 | 给网卡发送数据 |
| NET_RX_SOFTIRQ| 3 | 从网卡接收数据 |
| SCSI_SOFTIRQ  | 4 | SCSI命令的后中断处理 |
| TASKLET_SOFTIRQ| 5 | 处理常规tasklet |

这里的优先级就是软中断的索引，数值越小，代表优先级越高。Linux软中断处理程序总是从索引0开始执行。

# 2.1 软中断使用的数据结构

软中断的主要数据结构是`softirq_vec`数组，包含类型为`softirq_action`的32个元素。软中断的优先级表示`softirq_action`类型元素在数组中的索引。也就是说，目前只使用了数组中的前6项。`softirq_action`包含2个指针：分别指向软中断函数和函数使用的数据。

另一个重要的数据是`preempt_count`，存储在进程描述符中的`thread_info`成员中，用来追踪记录内核抢占和内核控制路径嵌套层数。它又被划分为4部分，如下表所示：

表4-10 `preempt_count`各个位域

| 位 | 描述 |
| -- | ---- |
| 0-7  | 内核抢占禁止计数（最大值255） |
| 8-15 | 软中断禁用深度计数（最大值255） |
| 16-27| 硬中断计数（最大值4096）|
| 28   | PREEMPT_ACTIVE标志     |

关于内核抢占的话题我们还会再写一篇专门的文章进行阐述，故在此不再详述。

可以使用宏`in_interrupt()`访问硬中断和软中断计数器。如果这两个计数器都是0，则返回值为0；否则返回非0值。如果内核没有使用多个内核态堆栈，该宏查找的是当前进程的`thread_info`描述符。但是，如果使用了多个内核态堆栈，则查找`irq_ctx`联合体中的`thread_info`描述符。在此情况下，内核抢占肯定是禁止的，所以该宏返回的是非0值。

最后一个跟软中断实现相关的数据是每个CPU都有一个32位掩码，用来描述挂起的软中断。存储在`irq_cpustat_t`数据结构的`__softirq_pending`成员中。对其具体的操作函数是`local_softirq_pending()`宏，用来是否禁止某个中断。

# 2.2 处理软中断

软中断的初始化使用`open_softirq()`函数完成，函数原型如下所示：

    void open_softirq(int nr, void (*action)(struct softirq_action *))
    {
        softirq_vec[nr].action = action;
    }

软中断的激活使用方法`raise_softirq()`，其参数是软中断索引`nr`，主要执行下面的动作：

1. 执行`local_irq_save`宏保存`eflags`寄存器中的IF标志并且禁止中断。

2. 通过设备CPU软中断位掩码的相应位将软中断标记为挂起状态。

3. 如果`in_interrupt()`返回1，直接跳转到第5步。如果处于这种情况, 要么是当前中断上下文中正在调用`raise_softirq()`、或者软中断被禁止。

4. 否则，调用`wakeup_softirqd()`唤醒`ksoftirqd`内核线程。

5. 执行`local_irq_restore`宏恢复IF标志。

应该周期性地检查挂起状态的软中断，但是不能因此增加太重的负荷。所以，软中断的执行时间点非常重要。下面是几个重要的时间点：

* 调用`local_bh_enable()`函数使能软中断的时候。

* 当`do_IRQ()`函数完成I/O中断处理，调用`irq_exit()`宏时。

* 如果系统中使用的是`I/O-APIC`控制器，当`smp_apic_timer_interrupt()`函数处理完一个定时器中断的时候。

* 在多核系统中，当CPU处理完一个由`CALL_FUNCTION_VECTOR`CPU间的中断引发的函数时。

* 当一个特殊的`ksoftirqd/n`内核线程被唤醒时。

# 2.3 do_softirq函数

如果某个时间点，检测到挂起的软中断（`local_softirq_pending()`非零），内核调用`do_softirq()`处理它们。这个函数执行的主要内容如下：

1. 如果`in_interrupt()`等于1，则函数返回。这表明中断上下文中正在调用do_softirq()函数，或者软中断被禁止。

2. 执行`local_irq_save`来保存IF标志的状态，并在本地CPU上禁用中断。

3. 如果`thread_union`等于4KB，如果有必要，切换到软IRQ堆栈中。

4. 调用`__do_softirq()`函数。

5. 如果在第3步切换到软IRQ堆栈，则恢复原来的堆栈指针到esp寄存器中，然后切换到之前使用的异常堆栈中。

6. 执行`local_irq_restore`恢复中断标志。

# 2.4 __do_softirq()函数

`__do_softirq()`函数读取位掩码，如果某位被置1，则执行其对应的处理函数。但是，在执行的过程中，可能会有新的软中断发生，这样后面的软中断处理就会延时。为了保证位掩码所有的软中断处理及时，`__do_softirq()`函数一次处理完所有的软中断。但是，这种机制又引发了新的问题，`__do_softirq()`函数一次运行时间过长。基于这个原因，`__do_softirq()`函数每次运行固定数量的循环次数，如果还有没执行的软中断，交给内核线程`ksoftirqd`进行处理。下面是`__do_softirq()`这个函数所做的工作：

1. 设置每次循环次数为10。

2. 拷贝软中断的位掩码到局部变量中。

3. 调用`local_bh_disable()`函数禁止软中断。

    为什么此时禁止软中断呢？因为执行那些可延时函数时，中断是处于使能状态的，意味着执行`__do_softirq()`函数的过程中，随时都会发生中断，那么立即响应中断，执行`do_IRQ()`函数。而`do_IRQ()`函数中，在最后会调用`irq_exit()`宏，这个宏会引发另一个调用` __do_softirq()`的程序执行。这在Linux内核中是禁止的，因为其可延时函数的执行都是串行的。所以，在此需要禁止软中断。

4. 清除正在执行的软中断对应掩码位。

5. 执行`local_irq_enable()`使能中断。

6. 执行软中断对应的函数。

7. 执行`local_irq_disable()`禁止中断。

8. 迭代次数减1，继续执行。

# 2.5 ksoftirqd内核线程

在较新的内核版本中，每个CPU都有自己的`ksoftirqd`内核线程。每个`ksoftirqd`内核线程调用`ksoftirqd()`函数，主要的执行内容如下所示：

    for(;;) {
        set_current_state(TASK_INTERRUPTIBLE);
        schedule();
        /* 处于TASK_RUNNING状态中 */
        while (local_softirq_pending()) {
            preempt_disable();
            do_softirq();
            preempt_enable();
            cond_resched();
        }
    }

该内核线程被唤醒后，会调用`local_softirq_pending()`检查软中断位掩码，如果必要检查`do_softirq()`函数。

也就是说，`ksoftirqd`内核线程是时间维度上的一种平衡策略。

软中断函数也可以重新激活自身。实际上，网络软中断和tasklet软中断就是这样做的。更重要的是，外部事件，比如网卡上的`数据包泛滥`也可以频繁地激活软中断。

连续大量的软中断会造成潜在的问题，引入内核线程也是为了解决这个问题。如果没有这个内核线程，开发者只能使用两种替代策略。

第一种策略就是正在执行软中断的时候忽略新的软中断。换言之，在执行`do_softirq()`函数的过程中，除了执行已经记录的挂起中的软中断之外，不会再检查是否还会发生软中断。这个方案有瑕疵，假设软中断函数在执行`do_softirq()`函数的过程中被重新被激活。最坏的情况就是，直到下一次定时器中断发生时，软中断不会被执行，即使当前处理器处于空闲状态。对于这种软中断延迟，网络开发者不可接受。

第二种策略就是`do_softirq()`函数持续地检查是否有挂起的软中断。只有当所有的软中断被处理完该函数才退出。这肯定满足了网络开发者的需求，但是对系统的普通用户却造成了很大的干扰：如果网卡的数据包流非常频繁，或者软中断函数保持自激活，`do_softirq()`函数就永远不会返回，用户态的程序实际上无法正常工作。

综上所述，`ksoftirqd`内核线程就是尝试解决这种很难抉择的问题。`do_softirq()`函数判断是否有软中断挂起。迭代一些次数后，如果还有软中断挂起，函数就会唤醒内核线程，自身终止，交给内核线程去处理后续的软中断。内核线程的优先级比较低，用户程序的执行不会受到影响。如果处理器处于空闲状态，挂起的软中断也会很快被执行。

# 3 Tasklet

Tasklet是I/O驱动中实现可延时处理函数的一种优选方法。Tasklet的实现基于两种软中断，分别为`HI_SOFTIRQ`和`TASKLET_SOFTIRQ`。多个tasklet可能对应相同的软中断，每个tasklet都有自己的处理函数。除了`do_softirq()`执行`HI_SOFTIRQ`的tasklet优先于` TASKLET_SOFTIRQ`之外，这两种软中断没有实质上的差异。

Tasklet和高优先级的tasklet分别存储在` tasklet_vec`和`tasklet_hi_vec`数组中。它们都包含与CPU（`NR_CPUS`）相同个数的元素，这些元素的类型是`tasklet_head`，也就是说tasklet描述符的管理还是通过链表的结构进行管理（由此可以看出，链表在Linux内核数据管理中的作用了）。tasklet描述符的数据结构是`tasklet_struct`，它的成员如下表所示：

表4-11 tasklet描述符的数据成员

| 名称 | 描述 |
| ---- | ---- |
| next | 指向下一个描述符 |
| state | tasklet的状态 |
| count | 锁计数 |
| func | 指向tasklet处理函数 |
| data | 一个tasklet函数可能使用的无符号长整形数 |

`state`包含两个标志：

* TASKLET_STATE_SCHED

    置1，表明tasklet正在挂起（也就是准备执行）。这意味tasklet描述符已经被插入到`tasklet_vec`和`tasklet_hi_vec`数组中了。

* TASKLET_STATE_RUN

    表明正在执行。对于单处理器系统，该标志没有使用。

假设你正在写一个设备驱动且想使用`tasklet`，需要做什么呢？

首先，你应该申请一个新的`tasklet_struct`数据结构并通过`tasklet_init()`完成初始化。`tasklet_init()`的参数为tasklet描述符地址，你的tasklet处理函数地址，还有可选的整数参数。

Tasklet可以通过`tasklet_disable_nosync()`或`tasklet_disable()`禁止。这两个函数都是增加tasklet描述符的count值。但是，后者必须等到正在运行的tasklet函数终止后才会返回。重新使能tasklet，使用`tasklet_enable()`函数。

为了激活tasklet，可以根据优先级分别调用`tasklet_schedule()`函数或者`tasklet_hi_schedule()`函数。它们的工作内容类似，如下所示：

1. 检查`TASKLET_STATE_SCHED`，如果设置，则返回（说明已经被调度过了）。

2. 调用`local_irq_save`保存中断标志IF并禁止中断。

3. 将tasklet描述符添加到`tasklet_vec[n]`或`tasklet_hi_vec[n]`数组中对应的列表的开始处，在此，n表示CPU的逻辑编号。

4. 调用`raise_softirq_irqoff()`激活`TASKLET_SOFTIRQ`或`HI_SOFTIRQ`软中断。

5. 调用`local_irq_restore`恢复中断标志IF。

接下来，我们看看tasklet是如何执行的。其实，跟其它软中断的执行过程类似。软中断被激活，`do_softirq()`就会执行对应的软中断函数。`HI_SOFTIRQ`软中断对应的函数为`tasklet_hi_action()`，而`TASKLET_SOFTIRQ`对应的函数为`tasklet_action()`。它们的执行过程也是类似的，如下所示：

1. 禁止中断。

2. 获取CPU的逻辑编号n。

3. 将tasklet描述符链表中的地址存储到局部变量链表中。

4. 清除`tasklet_vec[n]`或`tasklet_hi_vec[n]`数组中已经调度过的tasklet描述符列表。（赋值NULL即可）

5. 使能中断。

6. 遍历链表中的tasklet描述符：

    * 在多核处理器系统中，需要检查`TASKLET_STATE_RUN`标志。
    * 通过检查tasklet描述符中的count成员，判断是否被禁止。如果tasklet被禁止，清除`TASKLET_STATE_RUN`标志，重新将tasklet描述符插回到tasklet描述符链表中，然后再一次激活`TASKLET_SOFTIRQ`或`HI_SOFTIRQ`软中断。

    * 如果tasklet被使能，清除`TASKLET_STATE_SCHED`标志，然后执行tasklet对应的处理函数。

需要注意的是，除非tasklet函数激活自身。否则，一次激活只能触发一次tasklet函数的执行。