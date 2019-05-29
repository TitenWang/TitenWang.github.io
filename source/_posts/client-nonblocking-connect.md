---
title: 客户端非阻塞socket建链流程
date: 2016-10-23 15:36:07
tags:
  - Nginx
  - 网络编程
categories: Nginx
type: "categories"
---

　　TCP协议是面向连接的、可靠的、基于字节流的传输层协议。那使用tcp协议进行通信的两端是如何进行通信的？使用tcp协议进行通信的两端是通过套接字（scoket）来建立连接的。套接字socket主要有两种类型，阻塞和非阻塞。通常为了防止进程阻塞以及避免cpu被长时间占用，客户端和服务端一般都会采用非阻塞socket进行通信，其中Nginx就是一个典型的例子。下面我们就以Nginx的upstream机制所涉及的与后端服务器建链的流程来总结下使用非阻塞socket的客户端建链流程。<!-- more -->
　　先来看下Nginx的upstream机制所涉及的与后端服务器建链的流程，其相关函数为ngx_event_connect_peer，该函数的执行流程如下：
<div align="center"> {% asset_img ngx_event_connect_peer.png 图1 ngx_event_connect_peer %} </div>
　　在建链流程图中调用connect尝试与后端服务器建链那一步，如果connect返回0表明Nginx与后端服务器已经建立了连接，但是如果connect返回-1并且错误码是EINPROGRESS表明后端服务器由于某些原因暂时没有完成连接的建立，后续建立连接后会通知Nginx，此时Nginx的做法是，将连接对应的读写事件加入到了epoll中监控，并将写事件加入到定时器中，因为Nginx与后端服务器建立连接是有时间限制的，所以后续如果超时执行了写事件回调函数，则表示建链因超时而失败了。如果不是因为超时而是连接上有写事件发生调用事件回调函数，则会以SO_ERROR为参数调用setsockopt函数，然后根据错误码来判断Nginx是否与后端服务器建立了连接，这部分见在调用ngx_event_connect_peer函数的ngx_stream_proxy_connect函数中设置的读写事件回调函数ngx_stream_proxy_connect_handler。
　　从上面Nginx的处理流程中我们可以看到非阻塞socket客户端建链的一般步骤：
　　1、	调用socket接口获取一个socket描述符。
　　2、	根据实际情况设置socket的一些属性，如接收缓冲区和发送缓冲区的大小等等。
　　3、	将socket设置为非阻塞socket。通过调用哪个接口将socket设置为非阻塞的呢？可以以FIONBIO为参数调用ioctl函数将socket设置为非阻塞的，具体ioctl函数的描述和使用可以参见[Linux manual page](http://www.man7.org/linux/man-pages/man2/ioctl.2.html)。
　　4、	将socket描述符对应的EPOLLIN和EPOLLOUT事件加入到epoll的监控机制中，这样当socket描述符有事件发生时能够及时通知应用程序进行相应事件的处理。
　　5、	调用connect系统调用与后端服务器建立连接。如果connect函数返回0则表示应用程序与后端服务器建链成功，应用程序就可以执行下一步动作了；如果connect返回的是-1，那么就需要根据系统返回的错误码进一步分析导致调用connect出错的原因。在Linux下，如果错误码是EINPROGRESS表明与后端服务器只是暂时没有完成建链操作，并不代表失败，这个时候就需要设置socket描述符的EPOLLIN和EPOLLOU事件对应的处理函数。待后续socket写事件发生，epoll就会通知应用程序该socket目前可写，然后调用写事件对应的处理函数，判断建链是否成功，如果建链成功，则应用程序就可以执行下一步操作了，如果建链失败，则返回。除了EINPROGRESS之外的错误码都表示Nginx和后端服务器建链失败了。
　　上面的第五步中对于connect的调用为什么需要做如此细化的处理呢？这个就要看非阻塞socket的特性了，因为对于非阻塞的socket，connect系统调用会立即返回，如果返回0，表明建链应用程序与远端服务器建链成功，如果返回-1且错误码是EINPROGRESS，则会在后台进行三次握手的处理，后续可以通过select、epoll等I/O多路复用机制监控socket描述符的写事件来判断建链成功与否，在connect()函数的[Linux manual page](http://man7.org/linux/man-pages/man2/connect.2.html)中也有类似的描述，如下：
> EINPROGRESS
> 　　The socket is nonblocking and the connection cannot be completed immediately.
> 　　It is possible to select or poll for completion by selecting the socket for writing.
> 　　After select indicates writability, use getsockopt to read the SO_ERROR option at level 
> 　　SOL_SOCKET to determine whether connect() completed successfully (SO_ERROR is zero)
> 　　or unsuccessfully (SO_ERROR is　one of the usual error codes listed here, explaining 
> 　　the reason for the failure).
