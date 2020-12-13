title: Go netpoller网络模型

toc: true

date: 2020-12-13 10:15:18

categories: Go

tags: Go

---

使用Go进行网络编程是十分的高效和便捷的，在goroutine-per-connection这样的编程模式下开发者可以使用同步的模式来编写业务逻辑而无需关心网络的阻塞、异步等问题极大的降低了心智负担。同时Go基于 I/O multiplexing 和 goroutine scheduler 使得这个网络模型在开发模式和接口层面保持简洁的同时也具备较高的性能可以满足绝大多数的场景。

## I/O模型
在网络编程的世界里存在5种IO模型：

1. 阻塞 I/O (Blocking I/O)  
2. 非阻塞 I/O (Nonblocking I/O)
3. I/O 多路复用 (I/O multiplexing)
4. 信号驱动 I/O (Signal driven I/O)
5. 异步 I/O (Asynchronous I/O)

当应用程序对一个socket发起一次网络读写操作时分为两个步骤：

1. 等待网络数据流到来网卡可读(读就绪)/等待网卡可写(写就绪) ，将数据从网卡写入到内核/读取内核数据写入网卡
2. 从内核缓冲区复制数据到用户空间(读)/从用户空间复制数据到内核缓冲区(写)

同步IO与异步IO的主要区别在于执行第二步的时候应用程序是否需要同步等待。上述5种IO模型除了Asynchronous I/O是异步之外，其余都为同步模型。

至于阻塞与非阻塞主要区别在于第一步数据未到达时当前进程是否会被阻塞住。

在阻塞模式下当应用程序调用`recvfrom`读取socket时，如果数据未到达应用程序会被阻塞直到数据完全准备好。

在非阻塞模式下当应用程序调用`recvfrom`读取socket时，如果数据未准备好则内核会立即返回一个EWOULDBLOCK，应用程序接到一个 EAGAIN error。非阻塞模式下也可以是同步IO：基于轮询，当返回EAGAIN时应用程序继续发起`recvfrom`调用直到数据准备好。

当数据准备好之后就可以将数据从内核空间的缓存复制到用户空间的缓存，在复制数据这一步应用程序还是阻塞的。
![Non-blocking I/O](/img/non-blockingIO.jpg)


## I/O多路复用
IO多路复用是指一个线程同时监听多个IO连接的事件，阻塞等待，当某个连接发生可读写IO事件时收到通知。所以复用的是线程而不是IO连接。常见的多路复用选择器有：select/poll/epoll、kqueue、iocp

简单讲一下select/poll/epoll：

### select&poll
select主要存在以下几个缺点：

- 每次调用select都需要使用copy_from_user把fd集合从用户态拷贝到内核态，当fd很多时这个开销会很大
- 每次调用select都需要在内核遍历传递进来的所有fd挨个检查fd的状态，随着fd数量的增长 I/O 性能会线性下降
- 最大文件描述符数量限制，默认一次只能传入1024个fd，意味着一次只能监听1024个socket

poll在本质上与select没有区别只是扩大了可传入的fd集合的大小

### epoll
相比于select，用户可以通过epoll_ctl调用将fd注册到epoll实例上，而epoll_wait则会阻塞监听所有的epoll实例上所有的fd的事件。他解决了select未解决的两个问题：

1. 通过epoll_ctl注册fd，一个fd只完成一次从用户态到内核态的拷贝而不需要每次调用时都拷贝一次，并且epoll使用红黑树存储所有的fd因此重复注册是没用的
2. 当某个fd注册完成后会与对应的设备建立回调关系，当设备就绪触发中断后内核通过该回调函数将该fd添加到rdllist 双向就绪链表中。epoll_wait 就是去检查 rdllist 中是否有就绪的 fd，当 rdllist 为空时就会阻塞挂起当前调用epoll_wait进程，直到 rdllist 非空时进程才被唤醒并返回。此处epoll解决了select的第二个问题：不需要每次调用都遍历传进来的fd列表，从而不会因为随着fd数量的增多而性能下降。

## netpoller
首先对于epoll提供的三个调用接口Go对此都做了封装

```go
#include <sys/epoll.h>  
int epoll_create(int size);  
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

// Go 对上面三个调用的封装
func netpollinit()
func netpollopen(fd uintptr, pd *pollDesc) int32
func netpoll(block bool) gList
```

当调用`net.Listen`之后操作系统会创建一个socket同时返回与之对应的fd(被设置为非阻塞模式)，该fd用来初始化listener的netFD(go层面封装的网络文件描述符)，同时会调用`epoll_create`创建一个 epoll 实例 作为整个 runtime 的唯一 event-loop 使用。

当一个client连接server时，listener通过accept接受新的连接，该连接的fd被设置为非阻塞模式同时会起一个新的goroutine来处理新连接并将新连接对应的fd注册到epoll中。当goroutine调用`conn.Read`，`conn.Write`对连接进行读写操作遇到` EAGAIN`错误时会被 gopark 给park住进行休眠，让P去执行本地调度队列里的下一个可执行的G。这些被park住的goroutine会在goroutine的调度中调用`runtime.netpoll`被唤醒，然后调用 injectglist 把唤醒的 G 放入当前 P 本地调度队列或者全局调度队列去执行。

runtime.netpoll 的核心逻辑是： 根据入参 delay设置调用 epoll_wait 的 timeout 值，调用 epoll_wait 从 epoll 的 `eventpoll.rdllist`双向列表中获取IO就绪的fd列表，遍历epoll_wait 返回的fd列表， 根据调用`epoll_ctl`注册fd时封装的上下文信息组装可运行的 goroutine 并返回。(仔细看遍历fd列表的这个操作是不是又有点像select？)

调用`runtime.netpoll`的地方有两处：

- 在调度器中执行`runtime.schedule()`，该方法中会执行`runtime.findrunable()`，在`runtime.findrunable()`中调用了`runtime.netpoll`获取待执行的goroutine
- Go runtime 在程序启动的时候会创建一个独立的sysmon监控线程，sysmon 每 20us~10ms 运行一次，每次运行会检查距离上一次执行netpoll是否超过10ms，如果是则会调用一次`runtime.netpoll`

**敲黑板，划重点：**
上面过程中通过Listen和Accept返回的fd都被设置为非阻塞模式，原因是如果为阻塞模式则会使对应的G阻塞在system call上，此时与G对应的M也会被阻塞从而进入内核态，一旦进入内核态之后调度的控制权就不在go runtime手中也就无法借助go scheduler进行调度。在非阻塞模式下对应goroutine是被gopark给park住放入某个wait queue中，M可以继续执行下一个G。整个过程网络 I/O 的控制权都牢牢掌握在 Go 自己的 runtime 里。

## one more thing
在go1.14对timer进行了优化，其中一个优化点是将timer与netpoll结合。在go1.10之前由一个独立的timerproc通过小顶堆和futexsleep来管理定时任务，go1.10为了降低锁的竞争将其扩展为最多64个小顶堆和timerproc，但依然没有从本质上解决全局锁以及多次G、P和M间的切换导致的性能开销问题。

在go1.14中把存放定时事件的四叉堆放到P中，这样既很大程度上避免了全局锁又保证了G、P和M间的一致性避免相互之前多次切换，同时取消了timerproc协程转而使用netpoll来做就近时间的休眠等待：

1. 如果新添加的定时任务when小于netpoll的休眠等待时间`sched.pollUntil`就会激活netPoll的等待。也就是在`runtime.findrunable `里的最后会使用超时阻塞的方法调用`epoll_wait`，这样既监控了epoll实例红黑树上的fd，又可兼顾最近的定时任务
2. 在每次`runtime.schedule`调度时在`runtime.findrunable `中都会通过`checkTimers `来查找可运行的定时任务 

### 参考
[The Go netpoller](https://morsmachine.dk/netpoller)