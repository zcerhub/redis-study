为了学习redis的网络通信需要对c/cpp语言的网络编程有所了解，该篇通过一个简单的demo学习c/cpp网络编程中各个函数的用法

#### demo展示

该demo包含两个程序：server.cpp和client.cpp。server用于接受客户端的请求，然后向其响应Hello World。client根据ip和port向server发起网络请求得到服务端的响应结果。

server.cpp的内容如下：

```cpp
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main(){
    //创建套接字
    int serv_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);

    //将套接字和IP、端口绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));  //每个字节都用0填充
    serv_addr.sin_family = AF_INET;  //使用IPv4地址
    serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1");  //具体的IP地址
    serv_addr.sin_port = htons(1234);  //端口
    bind(serv_sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr));

    //进入监听状态，等待用户发起请求
    listen(serv_sock, 20);

    //接收客户端请求
    struct sockaddr_in clnt_addr;
    socklen_t clnt_addr_size = sizeof(clnt_addr);
    int clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_addr, &clnt_addr_size);

    //向客户端发送数据
    char str[] = "Hello World!";
    write(clnt_sock, str, sizeof(str));
   
    //关闭套接字
    close(clnt_sock);
    close(serv_sock);

    return 0;
}
```

编译命令：

```
root@74e2df854fac:~/demo# g++ server.cpp -o server
root@74e2df854fac:~/demo# ./server  //启动server
```

clent.cpp的内容如下：

```cpp
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
int main(){
    //创建套接字
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    //向服务器（特定的IP和端口）发起请求
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));  //每个字节都用0填充
    serv_addr.sin_family = AF_INET;  //使用IPv4地址
    serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1");  //具体的IP地址
    serv_addr.sin_port = htons(1234);  //端口
    connect(sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
   
    //读取服务器传回的数据
    char buffer[40];
    read(sock, buffer, sizeof(buffer)-1);
   
    printf("Message form server: %s\n", buffer);
   
    //关闭套接字
    close(sock);
    return 0;
}
```

执行命令：

```
root@74e2df854fac:~/demo#  g++ client.cpp -o client
root@74e2df854fac:~/demo# ./client   //启动client
Message form server: Hello World!
```

#### 函数介绍

#### 服务端

##### 1.创建socket对象

```
int socket (int __domain, int __type, int __protocol) 
```

linux系统是通过socket对象进行网络操作，所以首先通过socket()创建一个socket。该函数返回socket描述符。

```
int serv_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
```

- AF_INET：代表使用ipv4地址
- SOCK_STREAM：一个可靠的、面向连接的字节流
- IPPROTO_TCP ：使用TCP协议进行通信

##### 2.绑定socket和ip:port绑定

```
 bind (int __fd, __CONST_SOCKADDR_ARG __addr, socklen_t __len)
```

客户端是通过ip:port的方式访问服务端的，所以需要将socket绑定ip:port，该类使用sockaddr_in 封装ip和port

##### 3.监听socket

```
listen (int __fd, int __n) 
```

接受socket文件描述符上的连接

##### 4.接受客户端的请求

```
accept (int __fd, __SOCKADDR_ARG __addr,
		   socklen_t *__restrict __addr_len)
```

阻塞直到客户端的请求到来。此时会返回新创建的socket，使用该socket和client通信。

##### 5.服务端响应请求

```
write (int __fd, const void *__buf, size_t __n) 
```

向socket写入指定长度的数据。服务端通过该函数将需要响应的数据写到accept函数返回的socket中，这样client就可以接收了。

#### 客户端

##### 创建socket

```
socket (int __domain, int __type, int __protocol)
```

客户端的代码是：socket(AF_INET, SOCK_STREAM, 0)。基本上和服务端的参数一致。此处的__protocol=0，表示使用返回的socket进行通信。

##### 创建与服务端的连接

```
int connect (int __fd, __CONST_SOCKADDR_ARG __addr, socklen_t __len)
```

在指定的socket上打开连接。服务端接收到该连接后accept函数会被唤醒。

##### 客户端读取响应

```
read (int __fd, void *__buf, size_t __nbytes)
```

从指定描述符上读取数据到buf中。



#### 参考文献

- [一个简单的Linux下的socket程序](http://c.biancheng.net/cpp/html/3030.html)

