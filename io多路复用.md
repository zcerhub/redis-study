### IO多路复用

计算机可以通过select/poll/epoll/kqueue这些系统调用实现IO多路复用。实现了在一个线程中处理多个请求。

#### select

定义如下：

```
 int select (int __nfds, fd_set *__restrict __readfds,
		   fd_set *__restrict __writefds,
		   fd_set *__restrict __exceptfds,
		   struct timeval *__restrict __timeout);
```

工作原理：__readfds和_ _writefds为fd_set类型，代表用户监听指定fd的读事件和写事件。当某一个fd事件发生select函数返回。

缺点：

1. fd_set有大小限制（通常监控为1024）
2. 应用程序只有遍历_writefds、_ __readfds才能缺点发生事件的fd，事件复杂度为O(n)
3. 内核会将返回的事件保存到__readfds和 _writefds中，重新调用select需要重新设置
4. 每次调用都涉及到_writefds和 __readfds在用户态和内核态之间的复制，性能较差（问题的根源还是3）

#### poll

```
int poll (struct pollfd *__fds, nfds_t __nfds, int __timeout)


/* Data structure describing a polling request.  */
struct pollfd
  {
    int fd;			/* File descriptor to poll.  */
    short int events;		/* Types of events poller cares about.  */
    short int revents;		/* Types of events that actually occurred.  */
  };
```

优点：

1. 在结构体pollfd中保存监听的fd，不再传入固定长度的fd_set，从而解决了select的问题1
2. 在结构体中将感兴趣的事件和实际触发的事件分别保存到events和revents，从而解决了select的问题3

select的问题3、4仍旧没有解决。只要返回被触发事件的fd就可以解决问题2，在内核中保存fd需要监听的事件类型可以解决问题4.

#### epoll

```
   int epoll_create (int __size);
   
   int epoll_ctl (int __epfd, int __op, int __fd,
		      struct epoll_event *__event) ;
		      
   int epoll_wait (int __epfd, struct epoll_event *__events,int __maxevents, int __timeout);  
   
   
    typedef union epoll_data
    {
      void *ptr;
      int fd;
      uint32_t u32;
      uint64_t u64;
    } epoll_data_t;

    struct epoll_event
    {
      uint32_t events;	/* Epoll events */
      epoll_data_t data;	/* User data variable */
    } __EPOLL_PACKED;
```

- epoll_create：创建epoll对象

- epoll_ctl：对epoll对象的fd执行指定 EPOLL_CTL_*(EPOLL_CTL_ADD、EPOLL_CTL_DEL 、EPOLL_CTL_MOD)操作

- epoll_wait：等待epoll对象上监听的事件发生，返回触发的事件个数

#### kqueue

```
int kqueue(void);
int kevent(int kq, const struct kevent *changelist, int nchanges, struct kevent *eventlist, int nevents, const struct timespec *timeout);
```

kqueue和epoll实现基本一致，使用kevent代替了epoll_ctl、epoll_wait。在此不做过多赘述。

### epoll使用实例

以下为epoll官方给出的样例：

```c
#define MAX_EVENTS 10
struct epoll_event ev, events[MAX_EVENTS];
int listen_sock, conn_sock, nfds, epollfd;

/* Set up listening socket, 'listen_sock' (socket(),
  bind(), listen()) */

epollfd = epoll_create(10);
if(epollfd == -1) {
   perror("epoll_create");
   exit(EXIT_FAILURE);
}

ev.events = EPOLLIN;
ev.data.fd = listen_sock;
if(epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
   perror("epoll_ctl: listen_sock");
   exit(EXIT_FAILURE);
}

for(;;) {
   nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
   if (nfds == -1) {
       perror("epoll_pwait");
       exit(EXIT_FAILURE);
   }

   for (n = 0; n < nfds; ++n) {
       if (events[n].data.fd == listen_sock) {
           //主监听socket有新连接
           conn_sock = accept(listen_sock,
                           (struct sockaddr *) &local, &addrlen);
           if (conn_sock == -1) {
               perror("accept");
               exit(EXIT_FAILURE);
           }
           setnonblocking(conn_sock);
           ev.events = EPOLLIN | EPOLLET;
           ev.data.fd = conn_sock;
           if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock,
                       &ev) == -1) {
               perror("epoll_ctl: conn_sock");
               exit(EXIT_FAILURE);
           }
       } else {
           //已建立连接的可读写句柄
           do_use_fd(events[n].data.fd);
       }
   }
}
```

主要分为三部曲：

1. epoll_create创建epoll对象
2. epoll_ctl向epoll对象为fd执行添加、删除、修改操作
3. 在循环中通过epoll_wait获得触发的事件个数，然后依次处理

说明：

- accept事件处理

  listen_sock为server端的监听socket，如果该描述符的事件触发，说明发生了accept事件，则使用accept事件逻辑进行处理：accept方法返回客户端的conn_sock，使用epoll_ctl为epoll添加对conn_sock的 EPOLLIN | EPOLLET事件的监听（具体的代码为：epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock,&ev) )。

- read/write事件处理

  触发的fd不是listen_sock，说明在在accept阶段注册到client的socket，然后从events中获得索引n处的event进行处理。

##### 原理猜想

从代码分析来看events数组是应用程序传递给内核的，当监听的事件到来时，内核可以将事件监听的fd、事件类型及其传递的数据封装为epoll_event对象，将其保存到events数组，并返回此次触发事件的大小size。应用程序可以根据size和events获得此次触发的所有的监听事件信息，对其处理即可。

#### 参考文献

- [深入理解LinuxIO复用之epoll](https://segmentfault.com/a/1190000021369433)
- [Redis与Reactor模式](https://www.cnblogs.com/my_life/articles/5320230.html)