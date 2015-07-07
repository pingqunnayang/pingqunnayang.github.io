---
layout: post
title:  CSAPP学习笔记之网络编程
titlename:  CSAPP学习笔记之网络编程
date:  	2015-04-22 19:48:00  
category: 技术
tags: System Unix Socket
description:
---
&nbsp; &nbsp; &nbsp; &nbsp;不同计算机上的进程相互通讯的机制叫做网络进程间通讯（network IPC）。套接字是通讯端点的抽象，它与文件描述符访问文件一样，应用程序用套接字描述符访问套接字。
每个套接字都有相应的套接字地址，是有一个因特网地址和一个16位的整数端口组成的，用"地址：端口"来表示。当客户端发起一个连接请求时，客户端套接字地址中的端口是由内核自动分配的，称为`临时端口（ephemeral port）`，服务端套接字地址中的端口通常是某个知名的端口，和这个服务是相应的。
一个连接是由它两端的套接字地址唯一确认的。这对套接字地址叫做`套接字对（socket pair）`，由下列元组来表示：
`（cliaddr : cliport，servaddr : servport）`

![Alt text](/public/img/technology/unit_socket_1.png)

套接字接口（socket interface）是一组函数，它们和Unix I/O函数结合起来，用以创建网络应用,下面这张图就是引用各个接口建立连接的向导图。

![Alt text](/public/img/technology/unit_socket_2.png)

###1.1 创建套接字
创建一个套接字可以调用socket函数。

```c
	/* 成功返回非负描述符，若出错，返回-1 */
	int socket(int domain,int type,int protocol);
```

参数domain（域）确定通讯的特性，包括地址格式，如：AF_INET 表示IPV4因特网域，AF_INET6表示IPV6因特网域。
参数type确定套接字的类型，如：SOCK_RAW表示IP协议的数据报接口，SOCK_STREAM表示有序的、可靠的、双向的、面向连接的字节流。
参数Protocol通常是0，表示为给定的域和套接字类型选择默认协议。SOCK_STREAM的默认协议就是TCP。
###1.2 套字节与地址关联
&nbsp; &nbsp; &nbsp; &nbsp;客户端的套接字关联一个地址可以让系统选一个默认的地址，而服务端需要关联上一个总所周知的地址来接收客户端的请求，可以使用bind函数来关联地址和套接字。

```c
/* 成功返回0，出错返回-1 */
int bind(int sockfd,const struct sockaddr *ddr,socklen_t len)
``
&nbsp; &nbsp; &nbsp; &nbsp;一个地址标识一个特定通信域的套接字端点，地址的格式与这个特定的通讯域相关，为了使不同格式的地址能够传入到函数，地址会强制转换成一个通用的地址结构sockaddr，套字节可以自由地添加额外的成员并且定义sa_data成员的大小。

```c
struct sockaddr{
	sa_family_t sa_family;
	char sa_data[]; 
	...
}
```
对地址使用有一些限制
- 在进程正在运行的计算机上，指定的地址必须有效；不能指定一个其他机器的地址
- 地址必须和创建套接字时的地址族所支持的格式相匹配。
- 地址的端口必须大于1024。
- 一般只能将一个套接字端点绑定到一个给定的地址上，尽管有些协议允许多重绑定。
如果套接字已经绑定了，可以调用`getsocketname`来发现绑定到套接字上的地址，如果套接字与对方连接，可以用调用`getpeername`函数来找到对方的地址。
###1.3 建立连接
&nbsp; &nbsp; &nbsp; &nbsp;如果是出来一个面向连接的网络服务（SOCK_STREAM或SOCK_SEQPACKET)，那么在开始交换数据前，客户端进程套接字需要与服务端的进程套接字之间建立一个连接。

```c
/* 成功返回0，出错返回-1 */
int connnect(int sockfd,const struct sockaddr *addr,sicklen_t len)
```
如果sockfd没有绑定一个地址，connect会给调用者绑定一个默认地址。connect函数会阻塞，一直到连接成功建立或者发生错误，如果成功，sockfd描述符现在就准备好可以读写了。
###1.4 接收连接
&nbsp; &nbsp; &nbsp; &nbsp;客户端发起连接是发起连接请求的主动实体。服务器是等待来自客户端的连接请求的被动实体，默认情况下，内核会认为socket函数创建的描述符对应于主动套字节（active socket），服务器调用listen函数告诉内核，描述符是被服务器使用而不是被客户端使用，服务器调用listen函数宣告它愿意接受连接请求。

```c
/* 成功返回0，出错返回-1 */
int listen(int sockfd,int backlog)
```
参数backlog提示系统该进程所要入队的未完成连接请求数量。
listen函数将sockfd从一个主动套接字转化成一个监听套接字（listening socket），一旦有连接请求到来，可以使用accpet函数获的连接请求并建立连接。

```c
/*成功返回非负连接描述符，出错返回-1*/
int accept(int sockfd,struct sockaddr *addr,socklen_t *len)
```
&nbsp; &nbsp; &nbsp; &nbsp;accept函数等待来自客户端的连接请求到达侦听描述符listenfd，然后再addr中填写客户端的套接字地址，并返回一个已连接描述符（connected descriptor），这个描述符可被用来利用Unix I/O函数与客户端通讯。<br>
&nbsp; &nbsp; &nbsp; &nbsp;accept函数返回的套接字描述符连接到调用connect的客户端，它与监听描述符有很大的区别，传给accept的监听描述符只被创建一次，并存在于服务器的整个生命周期，而已连接描述符每次接收请求时都会创建一次。

![Alt text](/public/img/technology/unit_socket_3.png)

###1.5数据传输
&nbsp; &nbsp; &nbsp; &nbsp;既然一个套接字表示一个文件描述符，那么只要建立连接，就可以使用read和write来通过套接字通讯，但是想指定选项，从多个客户端接受数据包，或者发送带外数据，就需要使用6个为数据传递而设计的套接字函数。

```c
ssize_t send(int sockfd,const void *buf,size_t nbytes,int flags);
ssize_t recv(int sockfd,void *buf,size_t nbytes,int flags);
```
send函数类似于write，recv类似于函数read，两者最后一个参数都可以指定标志来改变出来传输数据的方式。（参考APUE16.5）
其他四个函数：`sendto`,`recvfrom`,这两个函数是指定发送地址和接收是得到数据发送者的源地址，`sendmsg`,`recvmsg`这两个函数是指定多重缓冲区传输数据。

> - 参考书籍
>- 《深入理解计算机系统》作者: Randal E.Bryant / David O'Hallaron 出版社: 中国电力出版社
>- 《UNIX环境高级编程》作者: W.Richard Stevens / Stephen A.Rago 出版社: 人民邮电出版社