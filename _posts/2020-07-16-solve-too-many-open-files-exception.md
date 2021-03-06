---
layout: post
title:  "排查Too many open files异常"
date:   2020-07-16 12:29:00 +0800
categories: Linux
tag: ['Linux']
---

* content
{:toc}

---

线上服务报文件打开过多报错， `Caused by: java.net.SocketException: Too many open files` 
```java
Caused by: java.net.SocketException: Too many open files
	at java.net.Socket.createImpl(Socket.java:460)
	at java.net.Socket.<init>(Socket.java:431)
	at java.net.Socket.<init>(Socket.java:244)
	at org.apache.flink.streaming.experimental.CollectSink.open(CollectSink.java:80)
	... 7 more
```
这是一个常见的异常，一般是系统设置 `open files`  过小，或者进程打开文件过多导致的。<br />首先检查系统参数
```java
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 128544
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 655350
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 128544
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```
发现 `open files` 设置正常，接下来需要排查是哪个进程出现文件打开过多或者句柄未释放问题
```java
lsof -n  |awk '{print$2}'|sort |uniq -c |sort -k 1 -n -r |head -n 10  
    652 23817
    620 22936
    388 22961
    308 8843
    132 31762
    120 3578
    116 9108    
```
第一列为文件句柄数，第二列为进程id，通过发现句柄数并没有超多多大限制，到底是哪里出了问题呢……<br />
<br />谜底揭晓：<br />一般线上机器账号只会开放相应应用的权限，不能查看所有的进程，需要通过root权限查看多有进程的状况。通过root账号果然发现出问题的进程。
```java
sudo lsof -n  |awk '{print$2}'|sort |uniq -c |sort -k 1 -n -r |head -n 10
    
```


