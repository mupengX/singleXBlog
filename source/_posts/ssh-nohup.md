title: ssh远程执行nohup命令不退出
toc: true
date: 2017-06-11 12:14:27
categories: bash
tags: [ssh,nohup]

---

## 现象

在本机执行`ssh target "nohup sh test.sh &"`,ssh并没有立即结束退出，而是等着test.sh执行完才退出，如果提前断开ssh则执行失败。使用`nohup`和`&`是想让test.sh在后台执行，并忽略`SIGHUP`信号，即使执行命令的console退出了，执行命令的进程也可以继续执行。而ssh远程执行nohup的命令不立即退出跟nohup没有太大的关系。将上面的命令换成下面的命令就会立即返回：
`ssh target "nohup sh test.sh >/dev/null 2&1 &"`

## 原因

ssh在执行远程的命令时，首先在远程的机器上建立一个sshd的进程，然后由这个sshd进程启动一个bash进程来执行传过来的命令。bash进程的0，1，2三个fd通过管道与sshd的相应的文件描述符关联起来。如果远程命令是后台执行会发现其父进程变为1，输入fd重定向到了/dev/null，但是输出和错误的fd是管道，与sshd进程关联，所以这次启动的sshd进程不会结束，必须重定向输出和错误fd。

对于命令：`ssh target "for i in 1 2 3;do sh ./test.sh;done"` 在targe机器上执行`ps -ef | grep 'test\|sshd'` 看到下面结果:

``` bash
work      14766     1  0 12:42 ?        00:00:00 /bin/bash .test.sh
root      18185  2387  0 16:24 ?        00:00:00 sshd: work [priv] 
work      18377 18185  0 16:24 ?        00:00:00 sshd: work@pts/0  
work      18406 18377  0 16:24 pts/0    00:00:00 bash -c for i in 1 2 3;do sh ./test.sh;done

```
由此可以看到进程的递进关系：sshd --> bash -c --> bash。所以本机执行`ssh target "cmd"`要能够立刻返回要满足：sshd进程没有要等着结束的子进程，这一点靠`&`来保证。其次，没有其他正在执行的进程与它有管道关系，这点靠重定向来保证。因为重定向只对单个简单命令或单个复合命令有效，对于组合的命令需要放到{}中：`ssh target "{ ./test.sh && ./test.sh; } >/dev/null 2>&1 &"`。

示例一：
`ssh target "for i in 1 2 3; do ./test.sh >/dev/null 2>&1 0</dev/null; done &"`
ssh不会立即退出，test.sh挨个执行，bash-c是后台执行但是1和2fd跟sshd有管道相连。

示例二：
`ssh target "for i in 1 2 3; do ./test.sh ; done >/dev/null 2>&1 &"`
ssh立即退出，test.sh挨个执行，执行test.sh的bash由bash-c启动，bash-c进行了重定向，所以子bash继承了重定向。

示例三：
`ssh target "for i in 1 2 3; do ./test.sh & done >/dev/null 2>&1 &"`
与上面的示例不同的是这里的test.sh不是顺序执行而是近似的同时执行。