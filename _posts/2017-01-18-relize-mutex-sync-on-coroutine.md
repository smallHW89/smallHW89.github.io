---
layout: post
title: 给协程加上同步互斥机制
---

#   给协程加上同步互斥机制
前面一篇文章介绍了Linux内的同步互斥的概念、内核态和用户态Linux提供的同步/互斥接口。这里本文介绍下如何给协程加上同步、互斥机制。

----

##  简单说下协程coroutine:
 [参考文章](http://blog.csdn.net/qq910894904/article/details/41699541)

操作系统的课本中对进程、线程的定义：进程是最小的资源分配单位，线程是最小的调度单位。
随着互联网的飞速发展，互联网后台Server服务通常要面临高请求、高并发的挑战，一些业务Server通常要面临很高的网络IO请求。这也就是C10K问题。
现在对C10K问题的解决方案已经很成熟了，主要是 非阻塞IO+IO复用(epoll,select等)+网络事件驱动，另外再配合多进程/多线程。
对这种非阻塞IO+IO复用+网络事件驱动的解决方案，我们通常称为异步模式，与之想对应的是同步模式。

#### 举个简单的例子：
对于服务srvA, 对于每个前端请求的逻辑如下：
收到前端Req，访问SrvB拉取数据，然后访问SrvC拉取数据，回包Rsp给前端

#### 对于同步模式的解决方案：
对每个前端Req都要有一个线程或者进程来处理，直到回包给前端，逻辑中的网络访过程通常用阻塞模式；无论是用线程池/进程池或者每个请求产生一个进程或者线程来处理，当前端请求量大时，系统中存在大量进程/线程时，线程的调度、占用的内核资源都是比较严重的问题。

#### 对于异步模式的解决方案：
利用IO复用+非阻塞IO，把会导致阻塞的操作转化为一个异步操作，主线程负责发起这个异步操作，并处理这个异步操作的结果。由于所有阻塞的操作都转化为异步操作，理论上主线程的大部分时间都是在处理实际的计算任务，少了多线程的调度时间，一个线程就能同时处理大量的客户端请求，所以这种模型的性能通常会比较好。但是这种模型，各种回调函数，一个前端请求的处理逻辑分散在代码的各个地方，开发维护成本高。

#### 近来兴起的协程解决方案：
协程能让原来要使用异步+回调方式写的非人类代码,可以用看似同步的方式写出来，性能却接近异步模式。
程是一种用户级的轻量级线程。协程拥有自己的寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存到其他地方，在切回来的时候，恢复先前保存的寄存器上下文和栈。因此：协程能保留上一次调用时的状态（即所有局部状态的一个特定组合），每次过程重入时，就相当于进入上一次调用的状态，换种说法：进入上一次离开时所处逻辑流的位置。
在并发编程中，协程与线程类似，每个协程表示一个执行单元，有自己的本地数据，与其它协程共享全局数据和其它资源。目前主流语言基本上都选择了多线程作为并发设施，与线程相关的概念是抢占式多任务（Preemptive multitasking），而与协程相关的是协作式多任务。

#### 关于线程、状态机模：
A Computer is a state machine. Threads are for people who can't program state machines。 意思大概是计算机其实本来就是个状态机，每个线程，进程都看以看作是一个拥有自己状态的实体，靠调度器来进行调度，各种硬件事件，时间时间来驱动每个实体状态的改变。具体可以网上查下。

##  几种支持协程的语言、库
1.  go-lang  多线程，每个线程里面多个协程，还可以对协程进行调度（从1个线程传递给另外一个线程）
2.  erlang
3.  lua 
4.  c/c++:  [libgo](https://github.com/yyzybb537/libgo) 支持多线程
5.  c/c++:   [微信libco](https://github.com/tencent-wechat/libco) 支持多线程
6.   boost库中的context , coroutine
7.   本人写的uthread, 是一个简单的例子，单进程单线程下的协程库。 
8.  上述语言、库中 有些实现了协程间的同步、互斥机制。比如golang的channel， libgo还提供了自旋锁等。
9.   PS : 大家可以看看上面关于协程的实现，里面协程的切换和Linux内核的进程/线程的调度切换类似：switch_to。 个人认为其实Linux的进程，线程，上面实现的协程的实现都是利用的状态机模型。

---

## 给协程实现同步互斥接口
笔者所在公司部门用的逻辑层框架中集成了协程，逻辑层通常提供RPC服务，每个前端请求包都要返回一个回包，业务逻辑一般是访问后端其他逻辑服务或者DB服务。
逻辑层框架对每个前端请求生成一个协程，开发者访问后端服务时，调用协程库的网络接口API完成协程调度（等待事件时切出，事件到达时唤醒），这样同步编码达到异步模型效果。

框架中的协程库是单进程，单线程的（如何利用多核机器，起多个进程即可），因为很少需要多个协程间的协作，而且协程的切换时机是开发者可控可知的，所以这个协程库并没有提供协程间的互斥、同步机制。（笔者自己实现的uthread例子，也是只支持单进程单线程的，目前没有实现同步互斥接口。）

最近工作中某些原因，于是尝试给部门的协程库加上同步、互斥接口：信号量； 互斥量； 条件变量;因为是单线程的不存在线程竞争所以这里实现起来比较简单

###  信号量：

```
/**
 *@brief 微线程的信号量 , 用于微线程间的同步
 */
class MtSem
{
    public:
        uint32_t  sem_seq( )
        {
            return _dwSemSeq;
        };
        friend  class MtFrame;
     private: 
        uint32_t  _dwSemSeq;     //信号量资源的唯一标识, Init时调度器分配
        int32_t  _dwSemInitValue;   //信号量的初始值，参考sem_init的第三个参数  
        int32_t  _dwSemCurValue;
        std::list<MicroThread *> _waitList;   //当资源消耗完时的 等待队列
};

int  MtFrame::SemInit(uint32_t  value )
{
    uint32_t seq = _nextSemSeq+1;
    _nextSemSeq++;
    if(seq == 0 )
    {
        seq++;
        _nextSemSeq++;
    }
    if(  _semMng.find(seq) != _semMng.end ()) 
    {
        MTLOG_ERROR("Init Sem Error  %d", errno);
        return 0; //资源消耗完
    }
    MtSem * sem = new MtSem;
    sem->_dwSemSeq = seq;
    sem->_dwSemInitValue = sem->_dwSemCurValue = value;
    _semMng[seq] = sem;
    return seq;
}

int MtFrame::SemWait(uint32_t semSeq  )
{ //等待
    if( _semMng.find(semSeq) == _semMng.end() )
    {
        MTLOG_ERROR("semSeq  %d not in semMng",semSeq);
        return -1;
    }
    MtSem * sem = _semMng[semSeq];
    if(sem == NULL )
    {
        MTLOG_ERROR("MtSem:%d is NULL ", semSeq);
        return -1;
    }

    if( sem->_dwSemCurValue <= 0 ) 
    {//挂住线程 
       MtFrame* mtframe = MtFrame::Instance();
       MicroThread* thread =mtframe->GetActiveThread();
        sem->_waitList.push_back(thread); 
        thread->sleep(0x7fffffff); //睡眠最大时间, 切走线程
        sem->_dwSemCurValue--; //且回来时肯定是_dwSemCurValue 已经>0了
    }
    else // _dwSemCurValue > 0
    {
        sem->_dwSemCurValue --;
    }
    return 0;
}
int MtFrame::SemPost(uint32_t semSeq )
{
    if( _semMng.find(semSeq) == _semMng.end() )
    {
        MTLOG_ERROR("semSeq  %d not in semMng",semSeq);
        return -1;
    }

    MtSem * sem = _semMng[semSeq];
    if(sem == NULL )
    {
        MTLOG_ERROR("MtSem:%d is NULL ", semSeq);
        return -1;
    }

    if(sem->_dwSemCurValue > 0 )
    { //没有线程挂住
       sem->_dwSemCurValue++;
    }

    else
    { //_dwSemCurValue <=0. 应该只能==0，不会小于0
      //有线程挂住，唤醒,唤醒几个呢？每次唤醒一个即可。因为也只增加了1个
       
        sem->_dwSemCurValue++; 
        if( sem->_waitList.size()> 0 )
        { //找到队列头部的线程，头部的先压入的，所以先唤醒
            MicroThread *thread = sem->_waitList.front();
            sem->_waitList.pop_front(); //从等待队列头删除
            //从调度器的睡眠列迁移到可运行队列
            RemoveSleep(thread);
            InsertRunable(thread);
        }
        else
        { //never run here
        }
    }
    return 0;
}

```

###  互斥量：

###  条件变量：  





