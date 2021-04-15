---
layout: post
title:  "不同的IO模型"
date:   2021-04-15 23:07:00 +0700
categories: [java]
---

## 前言
本文介绍五种IO模型的原理及作用, 包括同步阻塞、同步非阻塞、IO多路复用、信号驱动IO、异步非阻塞五种.

## 同步阻塞
示意图如下:
![avatar](/static/img/IO模型-同步阻塞.png)
如上所示, 同步阻塞io在发起调用之后不管有没有数据就绪都会等待直到数据就绪,copy到用户空间为止,
这种模型存在的问题就是我们在等待的时候无法做其他事情,浪费了cpu资源.

## 同步非阻塞
![avatar](/static/img/io模型-同步非阻塞.png)
如上所示, 同步非阻塞io在发起调用之后,如果数据没有就绪,那么内核会返回一个状态码而不会发生阻塞.
我们可以定时轮询,但是这会造成cpu资源的浪费.

## IO多路复用

### IO多路复用-select和poll
在上面的示例中, 我们管理的只有一个socket, 如果要同时处理多个socket的读/写事件, 那么我们可能会这样做:
将要监听的socket放到一个数组里, 使用一个while循环, 不断轮询遍历数组中的socket.  
但是这种方法会发起大量的系统调用, 会造成性能的大幅下降, 假如我们把遍历轮询socket这件事情交给内核去做,
而我们只需要把一个我们要监听的socket数组传给内核, 让内核遍历处理, 这样就省去了系统调用的损耗.  
实际上linux的select系统调用就是这个原理:
![avatar](/static/img/IO模型-IO多路复用-select.png)
select系统调用定义如下:
```c++
int select(
    int nfds,
    fd_set *readfds,
    fd_set *writefds,
    fd_set *exceptfds,
    struct timeval *timeout);
// nfds:监控的文件描述符集里最大文件描述符加1
// readfds：监控有读数据到达文件描述符集合，传入传出参数
// writefds：监控写数据到达文件描述符集合，传入传出参数
// exceptfds：监控异常发生达文件描述符集合, 传入传出参数
// timeout：定时阻塞监控时间，3种情况
//  1.NULL，永远等下去
//  2.设置timeval，等待固定时间
//  3.设置timeval里时间均为0，检查描述字后立即返回，轮询
```
服务端代码:
```c++
while(1) {
  connfd = accept(listenfd);
  fcntl(connfd, F_SETFL, O_NONBLOCK);
  fdlist.add(connfd);
}
```
一个线程不断accept连接, 并将接收到的socket放到list里.

```c++
while(1) {
  // 把一堆文件描述符 list 传给 select 函数
  // 有已就绪的文件描述符就返回，nready 表示有多少个就绪的
  nready = select(list);
  ...
}
```
另外一个线程不断调用select, 将socket列表传给内核, 但是当select返回时仍然要自己遍历
socket fd列表.
```c++
while(1) {
  nready = select(list);
  // 用户层依然要遍历，只不过少了很多无效的系统调用
  for(fd <-- fdlist) {
    if(fd != -1) {
      // 只读已就绪的文件描述符
      read(fd, buf);
      // 总共只有 nready 个已就绪描述符，不用过多遍历
      if(--nready == 0) break;
    }
  }
}
```
可见这种方法有如下问题:
- 每次调用select时都要将socket的fd列表作为参数复制到内核, 这种开销在fd数组很大时是惊人的.
- select调用只返回就绪的socket fd数量, 用户程序仍然要自己遍历找到就绪的socket.
- select在内核中还是通过轮询检查socket状态, 只不过没有系统调用的开销.


系统调用poll和select的作用相同, 与select的区别在于没有1024个文件描述符的限制.
```c++
int poll(struct pollfd *fds, nfds_tnfds, int timeout);

struct pollfd {
  intfd; /*文件描述符*/
  shortevents; /*监控的事件*/
  shortrevents; /*监控事件中满足条件返回的事件*/
};
```

### IO多路复用-epoll
epoll是如何解决上面的三个问题的呢?
在说明之前, 我们先来看epoll的用法:  
第一步, 先创建一个epoll句柄:
```c++
int epoll_create(int size);
```
第二步, 向句柄中添加、删除、修改要监听的文件描述符
```c++
int epoll_ctl(
  int epfd, int op, int fd, struct epoll_event *event);
```
第三步, 等待描述符就绪
```c++
int epoll_wait(
  int epfd, struct epoll_event *events, int max events, int timeout);
```

- 对于每次复制fd列表的问题, epoll只需要用户传一次socket fd就够了, 不需要每次调用都传.
- select调用之后需要自己遍历获得结果, epoll_wait会将就绪的描述符返回给用户, 而不需要每次遍历.
- select底层仍然是同步遍历, epoll底层是通知机制, 即在数据就绪时异步唤醒.

内部原理示意图:
![avatar](/static/img/IO模型-IO多路复用-epoll.png)














