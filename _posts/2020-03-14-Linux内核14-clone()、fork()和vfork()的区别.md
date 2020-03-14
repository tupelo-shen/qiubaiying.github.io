---
layout:     post
title:      Linux内核14-clone()、fork()和vfork()的区别
subtitle:   分析Linux内核中三个创建子进程的系统调用之间的差异
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

在分析这三个系统调用之前，先来理解一下进程的4要素:

* 执行代码

    每个进程或者线程都要有自己的执行代码，不论是独立的，还是共享的一段代码。

* 私有堆栈空间

    进程具有自己独立的堆栈空间，但是对于线程来说它是共享的。

* 进程控制块（task_struct）

    不论是进程还是线程，都有自己的task_struct。Linux对于线程的实现采用"轻进程"。

* 有独立的内存空间

    线程具有独立的内存空间，而线程共享内存空间。

Linux内核用于创建进程的系统调用有3个，它们的实现分别为：fork、vfork、clone。它们的作用如下表所示：

| 调用 | 描述 |
| --- | ---- |
| clone | 创建轻量级进程（也就是线程），pthread库基于此实现 |
| vfork | 父子进程共享资源，子进程先于父进程执行 |
| fork  | 创建父进程的完整副本 |


下面我们来看一下3个函数的区别：

# 1. clone()

创建轻量级进程，其拥有的参数是：

1. fn

    指定新进程执行的函数。当从函数返回时，子进程终止。函数返回一个退出码，表明子进程的退出状态。

2. arg

    指向fn()函数的参数。

3. flags

    一些标志位，低字节是表示当子进程终止时发送给父进程的信号，通常是SIGCHLD信号。其余的3个字节是一组标志，如下表所示：

    | 名称 | 描述 |
    | ------------- | -----------------------|
    | CLONE_VM      | 共享内存描述符和所有的页表 |
    | CLONE_FS      | 共享文件系统 |
    | CLONE_FILES   | 共享打开的文件 |
    | CLONE_SIGHAND | 共享信号处理函数，阻塞和挂起的信号等 |
    | CLONE_PTRACE  | debug用，父进程被追踪，子进程也追踪 |
    | CLONE_VFORK   | 父进程挂起，直到子进程释放虚拟内存资源 |
    | CLONE_PARENT  | 设置子进程的父进程是调用者的父进程，也就是创建兄弟进程 |
    | CLONE_THREAD  | 共享线程组，设置相应的线程组数据 |
    | CLONE_NEWNS   | 设置自己的命令空间，也就是有独立的文件系统 |
    | CLONE_SYSVSEM | 共享System V IPC可撤销信号量操作 |
    | CLONE_SETTLS  | 为轻量级进程创建新的TLS段 |
    | CLONE_PARENT_SETTID | 写子进程PID到父进程的用户态变量中 |
    | CLONE_CHILD_CLEARTID | 设置时，当子进程exit或者exec时，给父进程发送信号 |

4. child_stack

    指定子进程的用户态栈指针，存储在子进程的esp寄存器中。父进程总是给子进程分配一个新的栈。

5. tls

    指向为轻量级进程定义的TLS段数据结构的地址。只有CLONE_SETTLS标志设置了才有意义。

6. ptid

    指定父进程的用户态变量地址，用来保存新建轻量级进程的PID。只有CLONE_PARENT_SETTID标志设置了才有意义。

7. ctid

    指定新进程保存PID的用户态变量的地址。只有CLONE_CHILD_SETTID标志设置了才有意义。

clone()其实是一个C库中的封装函数，它建立新进程的栈并调用sys_clone()系统调用。sys_clone()系统调用没有参数fn和arg。事实上，clone()把fn函数的指针保存到子进程的栈中return地址处，指针arg紧随其后。当clone()函数终止时，CPU从栈上获取return地址并执行fn(arg)函数。

下面我们看一个C代码示例，看看clone()函数的使用：

    #include <stdio.h>
    #include <stdlib.h>
    #include <malloc.h>
    #include <linux/sched.h>
    #include <signal.h>
    #include <sys/types.h>
    #include <unistd.h>

    #define FIBER_STACK 8192

    int a;
    void *stack;

    int do_something()
    {
        printf("This is the son, and my pid is:%d,
            and a = %d\n", getpid(), ++a);
        free(stack);
        exit(1);
    }

    int main()
    {
        void * stack;
        a = 1;

        /* 为子进程申请系统堆栈 */
        stack = malloc(FIBER_STACK);

        if(!stack)
        {
            printf("The stack failed\n");
            exit(0);
        }
        printf("Creating son thread!!!\n");

        clone(&do_something, (char *)stack + FIBER_STACK, CLONE_VM|CLONE_VFORK, 0);//创建子线程
        printf("This is the father, and my pid is: %d, and a = %d\n", getpid(), a);

        exit(1);
    }

上面的代码就相当于实现了一个vfork()，只有子进程执行完并释放虚拟内存资源后，父进程执行。执行结果是：

    Creating son thread!!!
    This is the son, and my pid is:3733, and a = 2
    This is the father, and my pid is: 3732, and a = 2

它们现在共享堆栈，所以a的值是相等的。

# 2. fork()

linux将fork实现为这样的clone()系统调用，其flags参数指定为SIGCHLD信号并清除所有clone标志，child_stack参数是当前父进程栈的指针。父进程和子进程暂时共享相同的用户态堆栈。然后采用 **写时复制技术**，不管是父进程还是子进程，在尝试修改堆栈时，立即获得刚才共享的用户态堆栈的一个副本。也就是称为了一个单独的进程。

    #include <stdio.h>
    #include <stdlib.h>
    #include <sys/types.h>
    #include <unistd.h>

    int main(void)
    {
        int count = 1;
        int child;

        child = fork();
        if(child < 0){
            perror("fork error : ");
        }
        else if(child == 0){    // child process
            printf("This is son, his count is: %d (%p). and his pid is: %d\n",
                    ++count, &count, getpid());
        }
        else{                   // parent process
            printf("This is father, his count is: %d (%p), his pid is: %d\n",
                    count, &count, getpid());
        }
        return EXIT_SUCCESS;
    }

上面代码的执行结果：

    This is father, his count is: 1 (0xbfdbb384), his pid is: 3994
    This is son, his count is: 2 (0xbfdbb384). and his pid is: 3995

可以看出，父子进程的PID是不一样的，而且堆栈也是独立的（count计数一个是1，一个是2）。

# 3. vfork()

将vfork实现为这样的clone()系统调用，flags参数指定为SIGCHLD|CLONE_VM|CLONE_VFORK信号，child_stack参数等于当前父进程栈指针。

vfork其实是一种过时的应用，vfork也是创建一个子进程，但是子进程共享父进程的空间。在vfork创建子进程之后，阻塞父进程，直到子进程执行了exec()或exit()。vfork最初是因为fork没有实现COW机制，而在很多情况下fork之后会紧接着执行exec，而exec的执行相当于之前的fork复制的空间全部变成了无用功，所以设计了vfork。而现在fork使用了COW机制，唯一的代价仅仅是复制父进程页表的代价，所以vfork不应该出现在新的代码之中。

实际上，vfork创建的是一个线程，与父进程共享内存和堆栈空间：

我们看下面的示例代码：

    #include <stdio.h>
    #include <stdlib.h>
    #include <sys/types.h>
    #include <unistd.h>

    int main()
    {
        int count = 1;
        int child;
        printf("Before create son, the father's count is:%d\n", count);
        if(!(child = vfork())) {
            printf("This is son, his pid is: %d and the count is: %d\n",
                    getpid(), ++count);
            exit(1);
        }
        else {
            printf("This is father, pid is: %d and count is: %d\n",
                    getpid(), count);
        }
    }

执行结果为：

    Before create son, the father's count is:1
    This is son, his pid is: 3564 and the count is: 2
    This is father, pid is: 3563 and count is: 2

从运行结果看，vfork创建的子进程（线程）共享了父进程的count变量，所以，子进程修改count，父进程的count值也改变了。

另外由vfork创建的子进程要先于父进程执行，子进程执行时，父进程处于挂起状态，子进程执行完，唤醒父进程。除非子进程exit或者execve才会唤起父进程，看下面程序：

    #include <stdio.h>
    #include <stdlib.h>
    #include <sys/types.h>
    #include <unistd.h>

    int main()
    {
        int count = 1;
        int child;
        printf("Before create son, the father's count is:%d\n", count);
        if(!(child = vfork())) {

            int i;
            for(i = 0; i < 100; i++)
            {
                printf("Child process & i = %d\n", i);
                count++;
                if(i == 20)
                {
                    printf("Child process & pid = %d;count = %d\n",
                            getpid(), ++count);
                    exit(1);
                }
            }
        }
        else {
            printf("Father process & pid = %d ;count = %d\n",
                    getpid(), count);
        }
    }

执行结果为：

    Before create son, the father's count is:1
    Child process & i = 0
    Child process & i = 1
    Child process & i = 2
    Child process & i = 3
    Child process & i = 4
    Child process & i = 5
    Child process & i = 6
    Child process & i = 7
    Child process & i = 8
    Child process & i = 9
    Child process & i = 10
    Child process & i = 11
    Child process & i = 12
    Child process & i = 13
    Child process & i = 14
    Child process & i = 15
    Child process & i = 16
    Child process & i = 17
    Child process & i = 18
    Child process & i = 19
    Child process & i = 20
    Child process & pid = 3755;count = 23
    Father process & pid = 3754 ;count = 23

从上面的结果可以看出，父进程总是等子进程执行完毕后才开始继续执行。

# 总结

1. clone、vfork和fork是根据不同的需求而开发的。

    * clone 参数比较多，可以实现的控制就比较多，clone的设计初衷是给pthread线程库的开发提供支持的。其实用它也完全可以实现另外两种系统调用。
    * vfork是一个过时的系统调用，当时是因为写时复制（COW）技术还没有。所以才设计了这个子进程先于父进程的执行的创建进程的系统调用。
    * fork就是一个创建完整进程的调用。

2. clone、vfork和fork在内核层都是调用的_do_fork()这个函数。