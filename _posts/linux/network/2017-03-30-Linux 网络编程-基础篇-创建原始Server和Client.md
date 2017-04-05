---
layout: post
title:  "Linux 网络编程-基础篇-创建原始Server和Client"
date:   2017-03-30 14:36 +0800
categories: linux/network 
---
## getaddrinfo()

getaddrinfo() 返回的 struct addrinfo 相当于一个地址信息的总集合体，靠它一个就可以完成 socket/connect 或 socket/bind/listen/accept 的过程，避免了一些繁杂的地址结构体处理。

相对于自己创建 addrinfo 内的各种内容，我更喜欢使用 getaddrinfo()，但是 getaddrinfo 也并不是很简单的，需要梳理一下概念，帮助理解记忆：

```c
int getaddrinfo(const char *node, const char *service,
                const struct addrinfo *hints,
                struct addrinfo **res);
```

调用该函数的目的是：

进行 socket 编程时，我们要创建 socket 进行通信，不论是用于连接远端，还是用于接收远端的连接，都需要一个地址信息，这个信息在前一种情况下，是为了确定远端在网络中的位置，在后一种情况下，是为了确定自己在网络中的位置，以便别人过来连接。在创建的过程中，需要用到各种数据结构，addrinfo 的作用就是将它们都装在一起。

该函数的参数有三项：

1. node，该地址信息对应的 ip 地址；
2. service，对应的端口号；
3. hint 提示。因为 getaddrinfo 相当于一个地址筛选器，根据提供的参数，缩小符合的地址，最后返回符合条件的所有地址的 addrinfo，所以如果我们只提供 ip 和端口的话，就会返回太多的地址了（如 ipv4/ipv6/tcp/udp 的不同组合），所以还要在 hints 参数中，提供更多信息，缩小匹配范围，hint 本身就是 addrinfo，在里面填写更多信息。

该函数的返回的有两项：

1. int 返回值代表错误信息，为 0 时表示成功；
2. struct addrinfo **res 则是我们需要的 struct addrinfo 结构体的链表。

为何返回的是链表呢，因为函数会返回符合条件的所有地址的 addrinfo，保存在一个链表中。

为何需要传入二重指针呢，因为传入后 getaddrinfo 会改变它的值，它会申请并初始化链表，然后将第一个赋值给 res。因此，**当使用完毕返回的链表后，需要我们调用如下函数进行内存释放**：

```c
void freeaddrinfo(struct addrinfo *res);
```

addrinfo 的结构如下，res 指向第一个 addrinfo，其内部的 ai_next 指向链表中下一个 addrinfo：

```c
struct addrinfo {
    int              ai_flags;// 如果要创建监听 socket，则指定为 AI_PASSIVE
    int              ai_family;// address family，AF_INET/AF_INET6/AF_UNSPEC
    int              ai_socktype;// tcp/udp 
    int              ai_protocol;
    socklen_t        ai_addrlen;
    struct sockaddr *ai_addr;
    char            *ai_canonname;
    struct addrinfo *ai_next;
};
```

了解了 getaddrinfo() 的基本概念后，就可以开始创建客户端或服务器了。

## 创建 Client 客户端

> 本客户端的功能是，连接服务器，读取服务器传来的数据并打印，直到服务器关闭连接。

1. getaddrinfo()

   ```c
   struct addrinfo hints;
   struct addrinfo *result, *rp;

   memset(&hints, 0, sizeof(hints));
   hints.ai_family = AF_UNSPEC;//同时包含 ipv4 和 ipv6
   hints.ai_socktype = SOCK_STREAM;//要求使用 tcp socket

   if (getaddrinfo("127.0.0.1", "27777", &hints, &result) != 0) {

     printf("getaddrinfo error: %s\n", strerror(errno));
     return 0;
   }
   ```

   地址信息除了 ip 和端口外，我给予的 hint 还有 ai_family 和 ai_socktype，然后 getaddrinfo() 就会在 result 存放满足条件的 addrinfo 链表。

2. socket() 以及 connect()

   ```c
   int sockfd;

   for (rp = result; rp != NULL; rp = rp->ai_next) {

     if ((sockfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol)) == -1)
       continue;

     if (connect(sockfd, rp->ai_addr, rp->ai_addrlen) == -1) {

       printf("connect error: %s\n", strerror(errno));
       return 0;
     }
     else
       break; //成功一个后即跳出循环
   }

   //使用完成后释放 addrinfo
   freeaddrinfo(result);

   //一个都没有成功的情况
   if (sockfd < 0) {
     printf("invalid sockfd: %s\n");
     return 0;
   }
   ```

   由于 result 中得到的是链表，所以对链表进行逐节点扫描，注意 for 的书写形式，每次都从 ai_next 中获取下一个节点。

   在申请 socket 时，从 addrinfo 中取出数据即可。

   在进行 connect 时，亦从 addrinfo 中取出数据即可。

3. read()

   ```c
   #define MAXLINE 256
   int num = 0;
   char recvline[MAXLINE + 1];

   while ((num = read(sockfd, recvline, MAXLINE)) > 0) {

     recvline[num] = '\0';
     printf("%s\n", recvline);
   }

   if (num < 0) {
     printf("read error: %s\n", strerror(errno));
   }

   close(sockfd);
   ```

   从远端读取数据并打印，直到远端关闭连接。

## 创建 Server 服务器

> 本服务器的功能是，接收客户端的连接，并发送”Hello World!“，然后关闭连接。

1. getaddrinfo()

   ```c
   struct addrinfo hints;
   struct addrinfo *res, *rp;

   memset(&hints, 0, sizeof(hints));
   hints.ai_flags = AI_PASSIVE; //监听端口设置为 AI_PASSIVE
   hints.ai_family = AF_UNSPEC;
   hints.ai_socktype = SOCK_STREAM;

   if (getaddrinfo(NULL, "27777", &hints, &res) != 0) {

     printf("getaddrinfo error: %s\n", strerror(errno));
     return 0;
   }
   ```

   要求监听端口 27777，并且使用 tcp 协议。

2. socket() 以及 bind()

   ```c
   int listenfd = -1;

   for (rp = res; rp != NULL; rp = rp->ai_next) {

     listenfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);

     if (listenfd == -1)
       continue;

     if (bind(listenfd, rp->ai_addr, rp->ai_addrlen) == -1) {

       printf("bind error: %s\n", strerror(errno));
       return 0;
     }

     if (listenfd != -1)
       break;
   }

   //注意释放
   freeaddrinfo(res);
   ```

   通过 addrinfo 创建 socket 并 bind。

3. listen()

   ```c
   #define BACKLOG 7

   if (listenfd == -1) {

     printf("get socket error: %s\n", strerror(errno));
     return 0;
   }
   else {
     if (listen(listenfd, BACKLOG) == -1) {
       
       printf("listen error: %s\n", strerror(errno));
       return 0;
     }
   }
   ```

   监听端口。

4. write()

   ```c
   while (1) {

     sockfd = accept(listenfd, NULL, NULL);

     if (sockfd == -1) {

       printf("accept error: %s\n", strerror(errno));
       return 0;
     }

     write(sockfd, "Hello World!", strlen("Hello World!") + 1);

     close(sockfd);
   }
   ```

   向远端发送”Hello World!“后，关闭连接。

5. 概念梳理

   ### Socket Pair

   当服务器接收到新的连接，并通过 accept 返回新的 socket 时，其实新的 socket 还是使用的和服务器同一个端口，那么如何确定远端想和那个 socket 传输呢？ 

   socket pair {源 ip 地址：源端口，目的 ip 地址：目的端口}，一个 socket pair 唯一确定一个连接。

   ​