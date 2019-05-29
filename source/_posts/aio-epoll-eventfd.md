---
title: 异步文件I/O
date: 2016-09-03 15:45:01
tags:
  - Nginx
categories: Nginx
type: "categories"
---

　　Nginx中的文件异步I/O采用的不是glibc库提供的基于多线程实现的异步I/O,而是由Linux内核实现的。<!-- more -->Linux内核提供了5个系统调用来完成文件操作的异步I/O功能，列举如下：

| 方法名   | 参数含义  |  执行意义  |
| --- | :-----:  | :----:  |
| int io_setup(unsigned nr_events, aio_context_t *ctxp)     | nr_events表示需要可以处理的事件的最小个数，ctxp是文件异步I/O的上下文描述符指针 |   初始化文件异步I/O的上下文，返回0表示成功。|
| int io_destory(aio_context_t ctx)        |   ctx是文件异步I/O的上下文描述符   |   销毁文件异步I/O的上下文，返回0表示成功。   |
| int io_submit(aio_context_t ctx, long nr, struct iocb *cbp[])        |    ctx是文件异步I/O的上下文描述符，nr是一次提交的事件个数，cbp是提交的事件数组中的首个元素地址    |  提交文件异步I/O操作。返回值表示成功提交的事件个数  |
| int io_cancel(aio_context_t ctx, struct iocb *iocb, struct io_event *result)     | ctx表示文件异步I/O的上下文描述符，iocb是要取消的异步I/O操作，而result表示这个操作的执行结果 |   取消之前使用io_submit提交的一个文件异步I/O操作。返回0表示成功。     |
| int io_getevents(aio_context_t ctx, long min_nr,long_nr, struct io_event *events, struct timespec *timeout)        |   ctx表示文件异步I/O的上下文描述符，获取的已完成事件个数范围是[min_nr,nr]，events是执行完成的事件数组，timeout是超时时间，也就是获取min_nr个时间前的等待时间。   |   从已完成的文件异步I/O操作队列中读取操作。   |

　　文件异步I/O中有一个核心结构体--struct iocb，其定义如下：
```C
struct iocb {
	/*存储业务指针，与io_getevents方法返回的io_event结构体data成员一致*/
	u_int64_t aio_data;
	u_int32_t PADDED(aio_key, aio_reserved1);
	u_int16_t aio_lio_opcode;  //操作码
	int16_t aio_reqprio;  //请求的优先级
	u_int32_t aio_fildes;  //异步I/O操作的文件描述符
	u_int64_t aio_buf;  //读/写对应的用户态缓冲区
	u_int64_t aio_nbytes;  //读/写操作的字节长度
	int64_t aio_offset;  //读/写操作对应的文件中的偏移量
	u_int64_t aio_reserved2;
	/*
	 * 设置为IOCB_FLAG_RESFD,表示当有异步I/O请求完成时让内核
	 * 使用evenfd进行通知，这是与epoll配合使用的关键
	 */
	u_int32_t aio_flags;
	/*
	 * 当aio_flags为IOCB_FLAG_RESFD，用于事件
	 * 通知的eventfd句柄      
	 */
	u_int32_t aio_resfd;
}

```
　　上述的struct iocb结构体中，aio_flags和aio_read两个结构体成员正是Nginx中将文件异步I/O、eventfd以及epoll机制结合起来一起使用的关键，三者的结合也使得Nginx中的文件异步I/O同网络事件的处理一样高效。另外有一点需要说明的就是Nginx中目前只使用了异步I/O中的读操作，即struct iocb结构体中的成员aio_lio_opcode的值为IO_CMD_PREAD，因为文件的异步I/O不支持缓存操作，而正常写文件的操作往往是写入内存中就返回，而如果使用异步I/O方式写入的话反而会使得速度下降。
　　struct iocb用在提交和取消异步I/O事件中，而通过io_getevents获取已经完成的I/O事件时则用到的是另一个十分重要的结构体—struct io_event，其定义如下：
```C
struct io_event {
    uint64_t data;  //与提交事件时iocb结构体的aio_data成员一致
    uint64_t obj;  //指向提交事件时对应的iocb结构体
    /*异步I/O操作结果，res大于等于0表示成功，小于0失败*/
    int64_t res;
    int64_t res2;	//保留字段
}
```
　　在Nginx中，主要用到的字段就是data和res。其中data中保存的是文件异步I/O事件对象，res就是保存异步I/O的结果。
　　在简单了解了Linux内核提供的异步I/O系统调用及其在Nginx中涉及到的相关知识后，再来讲述下eventfd系统调用，因为这是Nginx将异步I/O事件集成到epoll中的一个桥梁。为什么这么说呢？通过上面对异步I/O结构体struct iocb结构体的分析，我们知道，当成员aio_flags设置为IOCB_FLAG_RESFD时，表明使用eventfd句柄来进行异步I/O事件的完成通知。正是这个eventfd，让其可以使用epoll机制来对其进行监控，从而间接对异步I/O事件完成进行监控，保证了事件驱动模块对网络事件和文件异步I/O事件的处理接口保持一致。
　　eventfd系统调用原型如下：
```C
int eventfd(unsigned int initval, int flags);
```
　　eventfd系统调用常用与进程之间通信或者用于内核与应用程序之间通信。在Nginx中正是利用了内核会在异步I/O事件完成时通过eventfd通知Nginx来完成对异步I/O事件的间接监控。
　　在简单介绍完Linux内核提供的异步I/O接口以及eventfd系统调用后，接下来开始分析文件异步I/O事件是如何在ngx_epoll_module中实现的。下面是涉及到的主要全局变量，可以分为两部分：
　　1)、系统调用相关：
　　int ngx_eventfd = -1; 这个是用于通知异步I/O事件完成的描述符，在Nginx中它会赋值给struct iocb结构体中的aio_resfd成员，也是epoll监控的描述符。
　　aio_context_t ngx_aio_ctx = 0; 这个就是异步I/O接口会使用到的异步I/O上下文，并且需要经过io_setup初始化后才能使用。
　　2)、与网络事件处理兼容相关。
　　static ngx_event_t ngx_eventfd_event; 这个就是eventfd描述符对应的读事件对象。因为文件异步I/O事件完成后，内核通知应用程序eventfd有可读事件(EPOLLIN)发生。然后应用程序就会调用读事件回调函数进行处理。
　　static ngx_connection_t ngx_eventfd_conn; 这个就是eventfd描述符对应的连接对象。
　　为什么要称它们是与网络事件处理兼容呢？回想下Nginx在处理网络事件的时候会为socket获取一个连接对象，然后设置连接对象ngx_connection_t的fd成员为socket描述符，接着设置连接的读事件和写事件，并设置对应的事件回调函数，最后将读/写事件（或整个连接）加入到epoll中监控。对应地处理文件异步I/O事件时，首先是让I/O事件的完成通知用eventfd来完成，然后设置eventfd的读事件及其处理函数，再用一个连接对象来保存eventfd和读事件，并将eventfd加入到epoll监控。这样就保证了Nginx内核可以像处理网络事件一样处理文件异步I/O事件。但是Nginx内核处理文件异步I/O事件又有其特别的地方。因为当epoll中监控到eventfd有读事件完成时，只可以说明Linux内核通知Nginx有文件异步I/O事件完成了，此时Nginx还并不知道有哪些或有几个异步I/O事件完成了，可以这么理解，eventfd仅仅是Linux内核用来通知Nginx有异步I/O事件完成了。那Nginx又是如何获取完成的异步I/O事件的呢，这就是eventfd描述符关联的读事件回调函数所需要完成的工作了，这个后面进行详细说明。
　　现假设有一个模块需要读取磁盘中的文件，那么如果Nginx启动了文件异步I/O处理的话，那么这个读盘的操作会被Nginx作为一个异步I/O事件来处理。
　　因为要将这个读盘事件以异步I/O方式来处理，那么首先就需要初始化一个异步I/O上下文，在Nginx中代码如下：
```C
static void
ngx_epoll_aio_init(ngx_cycle_t *cycle, ngx_epoll_conf_t *epcf)
{
    int  n;
    struct epoll_event  ee;

#if (NGX_HAVE_SYS_EVENTFD_H)
    ngx_eventfd = eventfd(0, 0); //调用eventfd()系统调用创建efd描述符
#else
    ngx_eventfd = syscall(SYS_eventfd, 0);
#endif
		……
    n = 1;
    /*设置ngx_eventfd为非阻塞*/
    if (ioctl(ngx_eventfd, FIONBIO, &n) == -1) {
        ……
    }
    /*
    *初始化文件异步io上下文，aio_requests表示至少可以处理的异步文件io事件
    *个数
    */
    if (io_setup(epcf->aio_requests, &ngx_aio_ctx) == -1) {
        ……
    }
    /*设置异步io完成时通知的事件*/
    /* ngx_event_t->data成员通常就是事件对应的连接对象*/
    ngx_eventfd_event.data = &ngx_eventfd_conn;
    ngx_eventfd_event.handler = ngx_epoll_eventfd_handler;
    ngx_eventfd_event.log = cycle->log;
    ngx_eventfd_event.active = 1;  //下面会加入到epoll中监控  
    ngx_eventfd_conn.fd = ngx_eventfd;
    ngx_eventfd_conn.read = &ngx_eventfd_event;
    /*文件异步io对应的连接对象读事件为ngx_eventfd_event*/
    ngx_eventfd_conn.log = cycle->log;

    ee.events = EPOLLIN|EPOLLET;  //监控eventfd读事件并设置为ET模式

    /*
     * ngx_eventfd被监控到有读事件发生时，会利用ee.data.ptr获取
     * 对应的连接对象,详见ngx_epoll_process_events()
     */
    ee.data.ptr = &ngx_eventfd_conn;

    /*
     * 将异步文件io的通知的描述符加入到epoll监控中，因为
     *在ngx_file_aio_read( )函数中将struct iocb结构体的aio_flags
     * 成员赋值为IOCB_FLAG_RESFD，他会告诉内核当异步io请求处理完成时使用
     *eventfd描述符通知应用程序，这使得异步io、eventfd和epoll可以结合起来
     *使用。另外，将将struct iocb结构体的aio_resfd设置为ngx_eventfd，
     *那么当有异步io事件完成时，epoll就会收到ngx_eventfd描述符的读
     *事件，然后*ngx_epoll_process_events()中会调用其读事件回调函数，即
     * ngx_epoll_eventfd_handler处理内核的通知。
     */
    if (epoll_ctl(ep, EPOLL_CTL_ADD, ngx_eventfd, &ee) != -1) {
        return;
    }
	……
}
```
　　通过调用ngx_epoll_aio_init方法，Nginx就将异步I/O以eventfd为桥梁与epoll结合起来了。
　　初始化完异步I/O上下文后，模块就可以提交文件异步I/O事件了。在此之前需要再了解下Nginx封装的一个异步I/O事件的对象，如下：
```C
struct ngx_event_aio_s {
    void  *data;
    /*由业务模块实现，用于在异步I/O事件完成后进行业务相关的处理*/
    ngx_event_handler_pt  handler;
    ngx_file_t   *file;  //文件异步I/O涉及的文件对象
    ……
    ngx_fd_t   fd;  //异步I/O将要操作的文件描述符
    ……
    /*aiocb就是struct iocb类型的，异步I/O事件控制块*/
    ngx_aiocb_t    aiocb;
    ngx_event_t    event;  //异步I/O对应的事件对象
};
```
　　那Nginx中是如何处理异步I/O事件的提交的呢？其代码实现如下：
```C
ssize_t
ngx_file_aio_read(ngx_file_t *file, u_char *buf, size_t size, off_t offset, ngx_pool_t *pool)
{
    ngx_err_t         err;
    struct iocb      *piocb[1];
    ngx_event_t      *ev;
    ngx_event_aio_t  *aio;
    ……
    /*
     * ngx_event_aio_t封装的异步io对象，如果file->aio为空，
     * 需要初始化* file->aio 
     */
    if (file->aio == NULL && ngx_file_aio_init(file, pool) != NGX_OK) {
        return NGX_ERROR;
    }
    aio = file->aio;
    ev = &aio->event;
    ……
    /*提交异步事件之前要初始化结构体struct iocb*/
    ngx_memzero(&aio->aiocb, sizeof(struct iocb));

    /*
     * 将struct iocb的aio_data成员赋值为异步io的事件对象，下面提交异步
     * I/O事件之后，等该事件完成，在通过io_getevents()获取到事件后，
     * 对应的struct io_event结构体中的data成员就会指向这个事件。
     * struct iocb的aio_data成员和struct io_event的data成员指向的是
     * 同一个东西
     */
    aio->aiocb.aio_data = (uint64_t) (uintptr_t) ev;
    aio->aiocb.aio_lio_opcode = IOCB_CMD_PREAD;
    aio->aiocb.aio_fildes = file->fd;
    aio->aiocb.aio_buf = (uint64_t) (uintptr_t) buf;
    aio->aiocb.aio_nbytes = size;
    aio->aiocb.aio_offset = offset;
    /*
     * 设置IOCB_FLAG_RESFD当内核有异步io请求处理完时
     * 通过eventfd通知应用程序
     */
    aio->aiocb.aio_flags = IOCB_FLAG_RESFD;  
    aio->aiocb.aio_resfd = ngx_eventfd;  //这个就是eventfd描述符

    /*
     * 将异步I/O对应的事件处理函数设置为ngx_file_aio_event_handler。
     *当io_getevents()函数中获取到该异步io事件时，会调用该回调函数，
     * 在Nginx中并不是直接调用，而是先将其加入到ngx_posted_event队列，
     * 等遍历完所有完成的异步io事件后，再依次调用所有事件的回调函数
     */
    ev->handler = ngx_file_aio_event_handler;

    piocb[0] = &aio->aiocb;

    /*
     * 将该异步io请求加入到异步io上下文中，等待io完成，内核会通过eventfd
     * 通知应用程序
     */
    if (io_submit(ngx_aio_ctx, 1, piocb) == 1) {
        ev->active = 1;
        ev->ready = 0;
        ev->complete = 0;

        return NGX_AGAIN;
    }
	……
}

```
　　在模块提交了文件异步I/O事件后，在事件完成之后，Linux就会触发eventfd的读事件来告诉Nginx异步I/O事件完成了，我们知道，当epoll监控到eventfd有事件发生时，在ngx_epoll_process_events()函数中会通过epoll_wait取出该事件，然后通过struct epoll_event结构体中的data.ptr成员获取eventfd对应的连接对象(在上面有介绍)，并调用连接对象中的读事件处理函数ngx_epoll_eventfd_handler()，而Nginx正是通过这个读事件处理函数来获取真正完成的文件异步I/O事件，这个读事件处理函数正是在ngx_epoll_aio_init()函数中进行注册的。该函数的实现如下：
```C
static void
ngx_epoll_eventfd_handler(ngx_event_t *ev)
{
    int               n, events;
    long              i;
    uint64_t          ready;
    ngx_err_t         err;
    ngx_event_t      *e;
    ngx_event_aio_t  *aio;
    struct io_event   event[64];  //一次性最多处理64个异步io事件
    struct timespec   ts;
    /*
    * 通过read获取已经完成的事件数，并设置到ready中，注意这里的ready
    * 可以大于64
    */
    n = read(ngx_eventfd, &ready, 8);
    ……
    ts.tv_sec = 0;
    ts.tv_nsec = 0;

    while (ready) {
        /*
         *从已完成的异步io队列中读取已完成的事件，返回值代表获取的事件个数
         */
        events = io_getevents(ngx_aio_ctx, 1, 64, event, &ts);
	    ……
        if (events > 0) {
            ready -= events;  //计算剩余已完成的异步io事件
            for (i = 0; i < events; i++) {
                /*
                 * data成员指向这个异步io事件对应着的实际事件，
                 * 这个与struct iocb 结构体中的aio_data成员是一致的。
                 * struct iocb 控制块中的aio_data成员被赋予对应io事件
                 * 对象是在函数ngx_file_aio_read()中实现的。
                 */
                e = (ngx_event_t *) (uintptr_t) event[i].data;
				  ……
                /*
                 * 异步io事件ngx_event_t->data成员指向的
                 * 就是ngx_event_aio_t对象，这个在
                 * ngx_file_aio_init()函数中可以看到
                 */
                aio = e->data;
                /*res成员代表的是异步io事件执行的结果*/
                aio->res = event[i].res;
                /*
                 * 将异步io事件加入到ngx_posted_events普通读写
                 * 事件队列中
                 */
                ngx_post_event(e, &ngx_posted_events);  
            }
            continue;
        }

        if (events == 0) {
            return;
        }
        /* events == -1 */
        ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_errno,
                      "io_getevents() failed");
        return;
    }
}	

```
　　一般来说业务模块在文件异步I/O事件完成后都需要在进行一些和业务相关的处理，那Nginx又是怎么实现的呢？当然是通过注册回调函数的方法来实现的。在通过ngx_epoll_eventfd_handler()回调函数获取到已经完成的文件异步I/O事件并加入到ngx_posted_events队列中后，执行ngx_posted_events队列中的事件时就会回调异步I/O事件完成的回调函数ngx_file_aio_event_handler(在ngx_file_aio_read注册)，然后在该函数中调用业务模块(提交文件异步I/O事件的模块)实现的回调方法，这个方法一般都是在业务模块提交异步I/O事件前注册到上面介绍的ngx_event_aio_t的handler成员中。其中异步I/O事件完成后的回调函数实现如下：
```C
static void
ngx_file_aio_event_handler(ngx_event_t *ev)
{
    ngx_event_aio_t  *aio;

    /*
     * 获取事件对应的data对象，即ngx_event_aio_t，这个在
     * ngx_file_aio_init()函数中初* 始化的
     */
    aio = ev->data;
    ……
    /*
     * 这个回调是由真正的业务模块实现的，举个例子如果是http cache模块，
     * 则会在ngx_http_file_cache_aio_read()函数中调用完
     * ngx_file_aio_read()后设置为ngx_http_cache_aio_event_handler()
     * 进行业务逻辑的处理，为什么要在调用完
     *  ngx_file_aio_read()之后再设置呢，因为可能业务模块一开始并没有为
     * ngx_file_t对* 象设置ngx_event_aio_t对象，而是在
     * ngx_file_aio_read()中调用ngx_file_aio_init()进行初始化的。
     */
    aio->handler(ev);
}
```
　　到这里Nginx中设计到的文件异步I/O、eventfd和epoll的配合使用就介绍完了。

**参考文章:**
　　1、《深入理解Nginx 模块开发与架构解析》
