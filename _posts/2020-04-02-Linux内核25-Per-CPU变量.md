---
layout:     post
title:      Linux内核25-Per-CPU变量
subtitle:   Per-CPU变量的设计思想及使用场景
date:       2020-04-02
author:     Tupelo Shen
header-img: img/post-bg-unix-linux.jpg
catalog:    true
tags:
    - Linux
    - Linux内核
    - 内核同步
    - Per-CPU
---


在前面的文章中我们已经学习了内核同步的一些基本概念，为什么需要内核同步
其实，最好的同步手段在于设计阶段就要尽量避免同步的需求。因为，毕竟同步的实现都是需要牺牲系统性能的。

既然多核系统中，CPU之间访问共享数据需要同步，那么最简单和有效的同步技术就是为每个CPU声明自己的变量，这样就减少了它们的耦合性，降低了同步的可能性。

**使用场景：**

一个CPU访问自己专属的变量，而无需担心其它CPU访问而导致的竞态条件。这意味着，`per-CPU`变量只能在特定情况下使用，比如把数据进行逻辑划分，然后分派给各个CPU的时候。

因为这些`per-CPU`变量全部元素都存储在内存上，所有的数据结构都会落在Cache的不同行上。所以，访问这些`per-CPU`变量不会导致对Cache行进行窥视（snoop）和失效（invalidate）操作，它们都对系统性能有很大的牺牲。

**缺点：**

1. 尽管，`per-CPU`变量保护了来自多个CPU的并发访问，但是无法阻止异步访问（比如，中断处理程序和可延时函数）。这时候，就需要其它同步技术了。

2. 此外，不管是单核系统还是多核系统，`per-CPU`变量都易于受到内核抢占所导致的竞态条件的影响。一般来说，内核控制路径访问每个CPU变量的时候，应该禁用内核抢占。假设，内核控制路径获得一个`per-CPU`变量的拷贝的地址，然后被转移到其它CPU上运行，这个值就可能会被其它CPU修改。

表5-3 列出了操作`per-CPU`变量的函数和宏

| 函数 | 描述 |
| ---- | ---- |
| DEFINE_PER_CPU(type, name)| 静态分配一个`per-CPU`数组 |
| per_cpu(name, cpu)        | 选择元素 |
| __get_cpu_var(name)       | 选择元素 |
| get_cpu_var(name)         | 禁止内核抢占，选择元素 |
| put_cpu_var(name)         | 使能内核抢占 |
| alloc_percpu(type)        | 动态分配一个`per-CPU`数组 |
| free_percpu(pointer)      | 释放动态分配的数组 |
| per_cpu_ptr(pointer, cpu) | 返回某个元素的地址 |