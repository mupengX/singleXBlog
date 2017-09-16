title: go upgrade on fly
toc: true
date: 2017-09-16 19:14:58
categories: Go
tags: Go

---

作为一个服务端程序在修改配置时最好能够支持优雅的重启，这样避免给用户带来糟糕的体验。即使不能优雅重启，那么在停止服务做配置修改时，也要做到优雅的停止，而不是粗暴的kill掉进程，这种粗暴的做法会lost正在处理的请求，对用户也是不友好的。

### 目标：
1. 不掐断当前处理中的连接
2. 不中断socket的监听
3. 新版本的程序能够接替老版本的程序

### 要做的事情：
1. 启动新的进程并监听到对应的address
2. 老进程停止接收新的连接
3. 老进程完成ongoing请求
4. 停止老进程

### 怎么做：
1. 通过Signal与进程交互，在程序中通过signal.Notify注册要接收的信号和接收信号的channel，向进程发送HUP signal通知其进行reload

2. go本身不支持端口复用，但是netListener支持从一个FD中开启监听。

3. 通过exec.Command来执行新的binary，子进程继承父进程的句柄，但是句柄里存在CloseOnExec问题，也就是在执行exec的时候，所有的句柄都关闭了，好在netListener.File()返回一个dup(2)的FD，该FD没有设置FD_CLOEXEC，通过Cmd的ExtraFile来传递FD。

4. 新启动的程序如何知道是从继承的FD监听还是新开一个socket？老进程在执行exec时设置一个环境变量，新进程通过该环境变量来进行判断

5. 停止老进程对端口的监听，调用老进程的netListener.Close()。该方法会使得所有被block住的Accept()方法不在被block住并返回一个error。

6. 等待老进程中ongoing的请求处理结束。这里使用sync.WaitGroup计数的方法，在Listener Accept()的时候Add，conn close时Done，在结束老进程时进行Wait()等待处理完所有的request。

fork-exec的方式存在一个问题是，新exec出来的是一个新的进程，如果用Systemd进行进程管理的话，Systemd会认为其已经dead了。为了避免这种情况需要维护一份pid文件，告诉Systemd PIDFile路径，Systemd通过该文件来监控。

除了fork-exec的方式外可以用master-worker的方式，master负责监听signal，worker负责处理请求。如果要进行升级master fork出新的worker转移env, listener, args等。程序还是那个程序，但是负责处理请求的进程则换成了新的worker进程

### 参考
[Graceful Restart in Golang](https://grisha.org/blog/2014/06/03/graceful-restart-in-golang/)

[Zero Downtime upgrades of TCP servers in Go](http://blog.nella.org/zero-downtime-upgrades-of-tcp-servers-in-go/)