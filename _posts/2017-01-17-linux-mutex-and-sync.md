---
layout: post
title: Linux中的同步互斥
---

## 互斥与同步的概念
互斥与同步两个概念比较容易混淆。
**互斥**：某一个资源同时只允许被一个访问者进行访问，具有唯一性和排他性。互斥无法限制访问者对资源的访问顺序。
**同步**：在互斥的基础上（大多数情况下），通过其他机制实现访问者对资源的有序访问。大多数情况下，同步已经实现了互斥。
**举个栗子**：
1个生产者、1个消费者的生产者消费者模型，这里生产者消费者之间要进行同步。
1个生产者，N个消费者的生产者消费者模型，这N个消费者之间要进行互斥。

## Linux内核中的同步互斥机制
**内核也需要支持互斥、同步：**现在计算机大多数是多处理器或者多核处理器，运行在不同处理器/核上的内核代码可能在同一时刻里面并发访问同一个共享数据。Linux内核现在是抢占式内核，调度器可以在任何时刻抢占正在运行的内核代码。
介绍抢占式内核的文章[Linux内核态抢占机制分析  ](http://mengren425.blog.163.com/blog/static/5690393120151625743841/)

### Linux内核提供的同步、互斥方法:
1. 原子操作，对简单的整型变量进行原子操作
2. 自旋锁，同一时刻只能被一个可执行线程持有，获得自旋锁时，如果被其他线程持有，则进行循环等待锁可用；自旋锁不可以递归。
3. 读写自旋锁，
4. 信号量：睡眠锁。一个任务试图获取信号量时，如果信号量未被占用，则该任务成功获取信号量； 如果该信号量已经被其他任务占用，则将该任务放入等待队列，让其睡眠，处理器继续工作，当该信号量被释放时，唤醒等待队列中的任务，并获取该信号量。
5. 读写信号量
6. 互斥体；只能在同一上下文中锁定，释放（信号量可以不是同一个上下文）
7. 完成变量，如果在内核中一个任务需要发出信号通知另一个任务发生了某个特定事件，使用完成变量去唤醒在等待的任务，使两个任务得以同步，信号量的简易方法。
8. BLK，大内核锁
9. 顺序锁

参考资料
[内核同步机制和实现方式](http://www.cnblogs.com/bastard/archive/2012/09/20/2694251.html)
[linux内核同步机制中的概念介绍和方法](http://blog.csdn.net/wealoong/article/details/7957385)
[全面解析Linux内核的同步与互斥机制--同步篇](http://blog.csdn.net/sailor_8318/article/details/2599357)

## Linux系统提供的进程、线程间同步互斥的机制
上面已经介绍了内核态中提供的同步互斥机制。在用户态中因为现在计算的多核、多CPU架构，以及用户态的抢占调度算法，这样用户态的代码可能同时并发访问一个共享数据。即使没有多核，多CPU，也可能因为抢占调度导致对同一个数据的竞争，比如2个线程都对全局变量i=i+1。 这里i=i+1对应汇编指令是多行，这样在执行到其中时可能被切到另外一个线程。

### Linux系统提供的进程\线程间同步互斥的接口:
1. 互斥量 pthread_mutex_t: 互斥量是线程同步的一种机制，用来保护线程的共享资源，同一时刻只允许一个线程对临界区进行访问。PS: 也能用于进程间的互斥，要和共享内存结合使用了，要设置进程共享属性
2. 条件变量：pthread_cond_t， 条件变量与互斥量不同，互斥量是防止多线程同时访问共享的互斥变量来保护临界区。条件变量是多线程间可以通过它来告知其他线程某个状态发生了改变，让等待在这个条件变量的线程继续执行。 条件变量要和互斥量结合使用。
3. 读写锁：pthread_rwlock_t。也可以用于进程间，要设置进程共享属性
4. 文件锁，记录锁： flock , fcntl [参考](http://blog.jobbole.com/104331/)
5. Posix 信号量：sem_open sem_close sem_wait, sem_post
6. SystemV信号量：semget semop semctl

## Linux 系统 的进程间通信机制
1. 管道pipe ,fifo
2. Posix消息队列：mg_open, mg_close ,mg_unlink, mg_send, mg_receive。 有同步的作用，可以用于生产者消费者模型
3. SystemV消息队列：msgget,msgsnd,msgrcv, msgctl
4. 共享内存：mamp, munmap, msync
5. Posix共享内存：shm_open, shm_unlink
6. SystemV共享内存:shmget, shmat, shmdt, shmctl
7. socket , unix socket, inet socket





 
 
 
 
