title: go 域名解析过慢
date: 2017-09-16 19:25:54
categories: Go
tags: Go
---

这几天发现一个问题，用go的HttpClient向`某个URL`发送post请求超时，经过一番排查，发现主要是在DNS解析的时候很慢要5S左右。但是同样的环境用Python和Java发送请求可以很快的返回结果，<1S。

首先看/etc/resolv.conf文件，默认配置了两个nameserver。strace执行过程发现在请求nameserver时有timeout，并且整个过程只请求了一个nameserver，同样的Python执行过程却没有这种情况。把resolv.conf中的option timeout从2设置为1之后，请求时间变为3S.如果增加一个nameserver：114.114.114.114，go的程序便可很快返回。

看了一下go关于域名解析的代码，觉得没毛病！ 可是为什么同样的环境下go跟Python的时间差这么多:( 而且目前只发现这个域名会出现这个问题。网上找了好久也没发现有人遇到这种问题。昨天跟leader讨论这个问题的时候，leader又提出用Java重写整个项目。心累，蓝瘦。
