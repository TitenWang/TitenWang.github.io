---
title: epoll的原理和实现
date: 2017-10-05 12:59:08
tags:
  - eBPF
  - 网络编程
  - 协议栈
categories: eBPF
type: "categories"
---
　　epoll是Linux内核为处理大批量文件描述符而设计的IO多路复用机制，它能显著提高程序在存在大量并发连接而只有少部分活跃连接情况下的系统CPU利用率。epoll之所以可以做到如此高的效率是因为它在获取就绪事件的时候，并不会遍历所有被监听的文件描述符集，而只会遍历那些被设备IO事件异步唤醒而加入就绪链表的文件描述符集。<!-- more -->
　　epoll机制提供了三个系统调用给用户态应用程序，用以管理感兴趣的事件，三个系统调用分别为epoll_create()、epoll_ctl()和epoll_wait()。另外还暴露了一个struct和一个union，分别为struct epoll_event和union epoll_data。而在epoll自身内部，还涉及了多个重要的数据结构，分别为struct eventpoll、struct epitem、struct eppoll_entry、struct ep_pqueue、struct poll_table_struct等。核心数据结构之间的关系如下图所示：
<div align="center"> {% asset_img epoll核心数据结构关系.png 图1 epoll核心数据结构关系 %} </div>
　　下面将从epoll机制提供给用户态的接口作为切入点，围绕核心数据结构，详细描述epoll的原理及其实现。
　　1、epoll_create()
　　epoll_create系统调用，其原型如下：
```c
	int epoll_create(int size)
```
　　epoll_create()系统调用会创建一个类型为struct eventpoll的对象，并返回一个与之对应文件描述符，之后应用程序在用户态使用epoll的时候都将依靠这个文件描述符，而在epoll内部则是通过该文件描述符进一步获取到struct eventpoll类型的对象，再进行对应的操作，这在其实现中可以很清晰看出。这个接口在内核中对应的实现为epoll_create1()，该函数位于eventpoll.c文件中，其实现如下：
```c
SYSCALL_DEFINE1(epoll_create1, int, flags)
{
	int error, fd;
	struct eventpoll *ep = NULL;
	struct file *file;
	……

	/* 创建struct eventpoll类型的对象 */
	error = ep_alloc(&ep);
	if (error < 0)
		return error;
	
	/* 获取一个空闲的文件描述符 */
	fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
	if (fd < 0) {
		error = fd;
		goto out_free_ep;
	}

	/*
	 * 创建一个名字为"[eventpoll]"的文件，并返回对应的struct file类型的对象,
	 * 从anon_inode_getfile()函数的实现可以看出，struct eventpoll类型的对象
	 * 被挂载到了struct file类型对象的private_data成员上。后续就可以通过
	 * struct file类型的对象获取struct eventpoll类型的对象。
	 */
	file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
				 O_RDWR | (flags & O_CLOEXEC));
	if (IS_ERR(file)) {
		error = PTR_ERR(file);
		goto out_free_fd;
	}
	ep->file = file;

	/* 
	 * 将文件描述符和文件对象关联起来，后续就可以通过fd获取struct eventpoll类型
	 * 的对象了，换句话说fd和struct eventpoll类型的对象关联起来了。
	 */
	fd_install(fd, file);
	return fd;

out_free_fd:
	put_unused_fd(fd);
out_free_ep:
	ep_free(ep);
	return error;
}
```
　　从实现中可以看到epoll_create()系统调用完成了fd、file、eventpoll三个对象之间的关联，并将fd返回给用户态应用程序，从而减小了用户态的使用难度。每一个epoll fd都会对应一个struct eventpoll类型的对象，该对象用来管理用户态应用程序添加的感兴趣事件以及就绪了的事件，对于struct eventpoll类型，其定义为：
```c
struct eventpoll {
	/* Protect the access to this structure */
	spinlock_t lock;
	struct mutex mtx;

	/* 用于收集调用了epoll_wait()系统调用的用户态应用程序 */
	wait_queue_head_t wq;

	/* Wait queue used by file->poll() */
	wait_queue_head_t poll_wait;

	/* 用于收集已经就绪了的item对象 */
	struct list_head rdllist;

	/* 用来挂载struct epitem类型对象的红黑树的根 */
	struct rb_root rbr;

	/*
	 * 这个单向链表也是用来收集就绪了item对象的，那这个成员什么时候会被使用呢?
	 * 这个成员是在对rellist成员进行扫描操作获取就绪事件返还给用户态时被用来存放
	 * 扫描期间就绪的事件的。为什么需要这样做呢?因为在对rellist扫描期间需要保证
	 * 数据的一致性，如果此时又有新的就绪事件发生，那么就需要提供临时的空间来存
	 * 储，所以ovflist就扮演了这个角色。
	 */
	struct epitem *ovflist;
	struct wakeup_source *ws;
	struct user_struct *user;

	/* struct eventpoll类型对象对应的文件对象 */
	struct file *file;
	int visited;
	struct list_head visited_list_link;
}
```
　　struct eventpoll类型的成员很多，到目前为止，我们只需要关注两个成员，一个是类型为struct rb_root的rbr，一个是类型为struct list_head的rdllist。其中rbr成员是一棵红黑树的根节点，这棵树中存放着所有通过epoll_ctl()系统调用添加到epoll中的事件对应的类型为struct epitem的对象；而rdllist链表则存放了将要通过epoll_wait()系统调用返回给用户态应用程序的就绪事件对应的struct epitem对象。
　　2、epoll_ctl()
　　epoll_ctl()系统调用，其原型如下：
```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
　　epoll_ctl()系统调用提供给用户态应用程序向epoll中添加、删除和修改感兴趣的事件，其中epfd就是通过epoll_create()系统调用获取的epoll对象文件描述符，op用于指明epoll如何操作event事件，其取值包括EPOLL_CTL_ADD、EPOLL_CTL_MOD和EPOLL_CTL_DEL，分别用于指明添加新的事件到epoll中、修改epoll中的事件以及删除epoll中的事件；fd就是被监控事件对应的文件描述符，而event则是被监控的事件对象，其类型为struct epoll_event，定义如下：
```c
struct epoll_event {
	__u32 events;
	__u64 data;
}
```
　　其中events成员指明了用户态应用程序感兴趣的事件类型，比如EPOLLIN和EPOLLOUT等。而data成员则是提供给用户态应用程序使用，一般用于存储事件的上下文，比如在nginx中，该成员用于存放指向ngx_connection_t类型对象的指针。
　　epoll_ctl()系统调用在内核对应的实现如下：
```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
		struct epoll_event __user *, event)
{
	int error;
	int full_check = 0;
	struct fd f, tf;
	struct eventpoll *ep;
	struct epitem *epi;
	struct epoll_event epds;
	struct eventpoll *tep = NULL;

	error = -EFAULT;
	/*
	 * 判断epoll是否需要从用户态拷贝事件，如果需要则拷贝，不需要直接往下走，如果
	 * 用户态传进来的op是删除事件操作，那么就不需要从用户态拷贝事件。
	 */
	if (ep_op_has_event(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		goto error_return;

	error = -EBADF;
	f = fdget(epfd);
	if (!f.file)
		goto error_return;

	/* 获取被监控的文件描述符对应的struct fd类型的对象 */
	tf = fdget(fd);
	if (!tf.file)
		goto error_fput;

	error = -EPERM;
	/* 被监控的文件描述符需要支持poll回调 */
	if (!tf.file->f_op->poll)
		goto error_tgt_fput;

	/* Check if EPOLLWAKEUP is allowed */
	if (ep_op_has_event(op))
		ep_take_care_of_epollwakeup(&epds);

	error = -EINVAL;
	/* epoll对象对应的fd不能被自己监控 */
	if (f.file == tf.file || !is_file_epoll(f.file))
		goto error_tgt_fput;
	……

	/* 
	 * 从文件对象的私有数据中获取struct eventpoll类型的对象，这个在epoll_create1()
	 * 函数中可以知道其挂载到了文件对象的私有数据成员中
	 */
	ep = f.file->private_data;
	……

	/* 尝试用fd和file对象从epoll的红黑树中去查找对应的struct epitem类型的对象 */
	epi = ep_find(ep, tf.file, fd);

	error = -EINVAL;
	switch (op) {
	case EPOLL_CTL_ADD:
		/* 
		 * 如果操作类型是add，且原先这个fd不在epoll中，那么就将其加入到epoll
		 * 中，如果fd已经在epoll中，则错误码设置为-EEXIST
		 */
		if (!epi) {
			epds.events |= POLLERR | POLLHUP;
			error = ep_insert(ep, &epds, tf.file, fd, full_check);
		} else
			error = -EEXIST;
		if (full_check)
			clear_tfile_check_list();
		break;
	case EPOLL_CTL_DEL:
		/* 
		 * 如果操作类型是del，并且fd对应的struct epitem类型的对象在epoll中，
		 * 那么就从epoll中移除
		 */
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;
	case EPOLL_CTL_MOD:
		/* 
		 * 如果操作类型是mod，并且fd对应的struct epitem类型的对象在epoll中，
		 * 那么就修改fd对应的struct epitem类型的对象感兴趣的事件类型。
		 */
		if (epi) {
			if (!(epi->event.events & EPOLLEXCLUSIVE)) {
				epds.events |= POLLERR | POLLHUP;
				error = ep_modify(ep, epi, &epds);
			}
		} else
			error = -ENOENT;
		break;
	}
	……

error_tgt_fput:
	if (full_check)
		mutex_unlock(&epmutex);

	fdput(tf);
error_fput:
	fdput(f);
error_return:
return error;
}
```
　　从epoll_ctl()的实现我们可以看到，在该函数中首先是通过epfd获取到对应的类型为struct eventpoll的epoll对象，接着判断epoll_ctl()参数fd对应的事件已经在epoll的监控中了，即用用户态传递进来的事件fd及其对应的file对象调用ep_find()到epoll对象的rbr成员中去寻找是否有对应的类型为struct epitem的对象，有则返回，否则返回NULL。然后再根据参数fd指定的操作类型对事件做进一步处理或进行异常处理。我们以事件之前未加入到epoll中，及操作类型EPOLL_CTL_ADD情况做进一步解析，在这种情况下会调用ep_insert()做进一步的处理，ep_insert()函数的实现如下：
```c
static int ep_insert(struct eventpoll *ep, struct epoll_event *event,
		     struct file *tfile, int fd, int full_check)
{
	int error, revents, pwake = 0;
	unsigned long flags;
	long user_watches;
	struct epitem *epi;
	struct ep_pqueue epq;

	/* 获取当前用户已经加入到epoll中监控的文件描述符数量，如果超过了上限，那么本次不加入 */
	user_watches = atomic_long_read(&ep->user->epoll_watches);
	if (unlikely(user_watches >= max_user_watches))
		return -ENOSPC;

	/* 从slab cache中申请一个struct epitem类型的对象 */
	if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
		return -ENOMEM;

	/* Item initialization follow here ... */
	INIT_LIST_HEAD(&epi->rdllink);
	INIT_LIST_HEAD(&epi->fllink);
	INIT_LIST_HEAD(&epi->pwqlist);
	/* 将epoll对象挂载到item的ep成员中 */
	epi->ep = ep;

	/* 设置被监控的文件描述符及其对应的文件对象到item的ffd成员中 */
	ep_set_ffd(&epi->ffd, tfile, fd);

	/* 保存fd感兴趣的事件对象 */
	epi->event = *event;
	epi->nwait = 0;
	epi->next = EP_UNACTIVE_PTR;
	if (epi->event.events & EPOLLWAKEUP) {
		error = ep_create_wakeup_source(epi);
		if (error)
			goto error_create_wakeup_source;
	} else {
		RCU_INIT_POINTER(epi->ws, NULL);
	}

	/* Initialize the poll table using the queue callback */
	epq.epi = epi;
	
	/* 初始化struct poll_table类型的对象 */
	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

	/* 返回值表示该文件描述符感兴趣的事件。如果感兴趣的事件没有发生，则为0 */
	revents = ep_item_poll(epi, &epq.pt);

	error = -ENOMEM;
	if (epi->nwait < 0)
		goto error_unregister;

	/* Add the current item to the list of active epoll hook for this file */
	spin_lock(&tfile->f_lock);
	/* 将item对象挂载到对应的文件对象的f_ep_links成员中 */
	list_add_tail_rcu(&epi->fllink, &tfile->f_ep_links);
	spin_unlock(&tfile->f_lock);

	/* 将item对象插入到epoll对象的红黑树中 */
	ep_rbtree_insert(ep, epi);

	/* now check if we've created too many backpaths */
	error = -EINVAL;
	if (full_check && reverse_path_check())
		goto error_remove_epi;

	/* We have to drop the new item inside our item list to keep track of it */
	spin_lock_irqsave(&ep->lock, flags);

	/* If the file is already "ready" we drop it inside the ready list */
	/*
	 * revents & event->events不为0，说明当前fd有感兴趣的事件发生。如果fd对应的item
	 * 对象又不在ready list中，那么就将item对象加入到epoll对象的ready list链表中，
	 * 表示事件就绪。
	 */
	if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
		list_add_tail(&epi->rdllink, &ep->rdllist);
		ep_pm_stay_awake(epi);

		/*
		 * 如果有应用程序在等待事件发生，那么就唤醒上层等待事件发生的应用程序，
		 * 那ep->wq这个成员是在什么时候被添加了等待epoll事件发生的应用程序信息
		 * 的呢?这个就是通过epoll_wait()系统调用来感知的。当有应用程序调用了
		 * epoll_wait()的时候，在ep_poll()函数中就会把当前应用程序current对象
		 * 加入到ep->wq成员中。
		 */
		if (waitqueue_active(&ep->wq))
			wake_up_locked(&ep->wq);
		if (waitqueue_active(&ep->poll_wait))
			pwake++;
	}

	spin_unlock_irqrestore(&ep->lock, flags);

	/* 增加当前用户加入epoll中的文件描述符个数 */
	atomic_long_inc(&ep->user->epoll_watches);

	/* We have to call this outside the lock */
	if (pwake)
		ep_poll_safewake(&ep->poll_wait);

	return 0;

error_remove_epi:
	spin_lock(&tfile->f_lock);
	list_del_rcu(&epi->fllink);
	spin_unlock(&tfile->f_lock);

	rb_erase(&epi->rbn, &ep->rbr);

error_unregister:
	ep_unregister_pollwait(ep, epi);

	spin_lock_irqsave(&ep->lock, flags);
	if (ep_is_linked(&epi->rdllink))
		list_del_init(&epi->rdllink);
	spin_unlock_irqrestore(&ep->lock, flags);

	wakeup_source_unregister(ep_wakeup_source(epi));

error_create_wakeup_source:
	kmem_cache_free(epi_cache, epi);

	return error;
}
```
　　从ep_insert()函数的实现我们可以看到，ep_insert()函数主要做了如下三件事：
　　1）、创建并初始化一个strut epitem类型的对象，完成该对象和被监控事件（包括fd、struct epoll_event类型的对象）以及epoll对象eventpoll的关联。
　　2）、将struct epitem类型的对象加入到epoll对象eventpoll的红黑树中管理起来。
　　3）、将struct epitem类型的对象加入到被监控事件对应的目标文件的等待列表中，并注册事件就绪时会调用的回调函数，在epoll中该回调函数就是ep_poll_callback()。
　　第一和第二点比较清晰，下面将着重分析第三点的流程。
　　在ep_insert()函数中，epoll会定义一个类型为struct ep_pqueue的对象，该对象包括了epitem成员，以及一个类型为poll_table的对象成员pt。在ep_insert()函数中我们会将 pt的_qproc这个回调函数成员设置为ep_ptable_queue_proc()，并在ep_item_poll()函数中将pt的_key成员设置为用户态应用程序感兴趣的事件类型，然后调用被监控的目标文件的poll回调函数。被监控的目标文件的poll回调函数一般会调用poll_wait()函数，而poll_wait()又会调用pt的_qproc()回调函数，而在ep_insert()函数中设置的pt的_qproc()回调函数ep_ptable_queue_proc()会将epitem对象对应的eppoll_entry对象加入到被监控的目标文件的等待队列中，并设置感兴趣事件发生后的回调函数为ep_poll_callback()。目标文件的poll回调函数调用完poll_wait()之后会获取对应的就绪事件掩码。如果pt的回调函数成员_qproc没有设置，那么目标文件的poll回调函数一般就只会返回对应的就绪事件掩码。所以如果目标文件对应的事件就绪的话，ep_item_poll()函数就会返回。ep_item_poll()在epoll_ctl()和ep_wait()的处理流程中都会调用，两个流程中的区别在于epoll_ctl()处理流程中调用ep_item_poll()函数的是时候会设置poll_table的_qproc成员；而在epoll_wait()处理流程中则不会设置该成员，而只会获取就绪事件的掩码。
　　以socket为例，因为socket有多种类型，如tcp、udp等，所以socket层会实现一个通用的poll回调函数，这个函数就是sock_poll()。在sock_poll()函数中通常会调用某一具体类型协议对应的poll回调函数，以tcp为例，那么这个poll()回调函数就是tcp_poll()。当socket有事件就绪时，比如读事件就绪，就会调用sock->sk_data_ready这个回调函数，即sock_def_readable()，在这个回调函数中则会遍历socket 文件中的等待队列，然后依次调用队列节点的回调函数，在epoll中对应的回调函数就是ep_poll_callback()，这个函数会将就绪事件对应的epitem对象加入到epoll对象eventpoll的就绪链表rdllist中，这样用户态程序调用epoll_wait()的时候就能获取到该事件，如果调用ep_poll_callback()函数的时候发现epoll对象eventpoll的ovflist成员不等于EP_UNACTIVE_PTR的话，说明此时正在扫描rdllist链表，这个时候会将就绪事件对应的epitem对象加入到ovflist链表暂存起来，等rdllist链表扫描完之后在将ovflist链表中的内容移动到rdllist链表中，此部分实现可以参考ep_scan_ready_list()函数。
　　3、epoll_wait()
　　epoll_wait()系统调用，其原型如下：
```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```
　　epoll_wait()系统调用主要是用于收集在epoll中监控的就绪事件。epoll_wait()函数返回值表示的是获取到的就绪事件个数，epfd表示的epoll对象fd，第二个参数则是已经分配好内存的epoll_event结构体数组，用于给内核存放就绪事件的；第三个参数表示本次最多可以返回的就绪事件个数，这个通常和events数组的大小一样；第四个参数表示在没有检测到事件发生时epoll_wait()的阻塞时长。epoll_wait()系统调用在内核对应的实现如下：
```c
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
		int, maxevents, int, timeout)
{
	int error;
	struct fd f;
	struct eventpoll *ep;

	/* The maximum number of event must be greater than zero */
	if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)
		return -EINVAL;

	/* Verify that the area passed by the user is writeable */
	if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event)))
		return -EFAULT;

	/* 通过epfd获取到对应的struct file类型的文件对象 */
	f = fdget(epfd);
	if (!f.file)
		return -EBADF;

	error = -EINVAL;
	/* 判断用户态传进来的fd是不是epfd，即通过判断文件对象的操作回调是不是eventpoll_fops */
	if (!is_file_epoll(f.file))
		goto error_fput;

	/* 
	 * 在epoll_create1()函数中我们知道epoll对象会存放在其对应的文件对象的私有
	 * 数据成员中，所以这里可以通过这个成员获取到epoll对象。
	 */
	ep = f.file->private_data;

	/* 获取就绪的事件，返回给用户态应用程序 */
	error = ep_poll(ep, events, maxevents, timeout);

error_fput:
	fdput(f);
	return error;
}
```
　　从epoll_wait()对应的内核实现我们可以看到，epoll_wait()首先是根据epfd获取到epoll对象eventpoll然后再调用ep_poll()获取就绪事件，也就是说ep_poll()函数是真正完成就绪事件获取工作的，其实现如下：
```c
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, long timeout)
{
	int res = 0, eavail, timed_out = 0;
	unsigned long flags;
	u64 slack = 0;
	wait_queue_t wait;
	ktime_t expires, *to = NULL;

	/* 
	 * 如果用户态传进来的epoll_wait()的阻塞时间大于0，则换算超时时间。如果一直
	 * 没有就绪事件发生，那么epoll_wait()就会休眠，让出处理器，等超时时间到了才
	 * 返回;如果有就绪事件就先放到用户态内存中，然后会返回用户态。
	 */
	if (timeout > 0) {
		struct timespec64 end_time = ep_set_mstimeout(timeout);

		slack = select_estimate_accuracy(&end_time);
		to = &expires;
		*to = timespec64_to_ktime(end_time);
	} else if (timeout == 0) {
		/* 如果用户态传进来的timeout等于1，那么如果没有就绪事件就立即返回 */
		timed_out = 1;
		spin_lock_irqsave(&ep->lock, flags);
		goto check_events;
	}

fetch_events:
	spin_lock_irqsave(&ep->lock, flags);

	if (!ep_events_available(ep)) {
		/* 
		 * current宏返回的是thread_info结构task字段,task正好指向与thread_info
		 * 结构关联的那个进程描述符。这里把调用了epoll_wait()系统调用等待epoll		     
		 * 事件发生的进程加入到ep->wq等待队列中。并设置了默认的回调函数用于唤
		 * 醒应用程序。
		 */
		init_waitqueue_entry(&wait, current);
		__add_wait_queue_exclusive(&ep->wq, &wait);

		for (;;) {
			set_current_state(TASK_INTERRUPTIBLE);
			/* 有就绪事件，或者超时则退出循环 */
			if (ep_events_available(ep) || timed_out)
				break;
			if (signal_pending(current)) {
				res = -EINTR;
				break;
			}

			spin_unlock_irqrestore(&ep->lock, flags);
			/* 休眠让出处理器 */
			if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
				timed_out = 1;

			spin_lock_irqsave(&ep->lock, flags);
		}

		/* 从ep->wq等待队列中将调用了epoll_wait()的进程对应的节点移除 */
		__remove_wait_queue(&ep->wq, &wait);
        
		/* 设置当前进程的状态为RUNNING */
		__set_current_state(TASK_RUNNING);
	}
check_events:
	/* 
	 * 判断epoll对象的rdllist链表和ovflist链表是否为空，如果不为空，说明有就绪
	 * 事件发生，那么该函数返回1，否则返回0
	 */
	eavail = ep_events_available(ep);

	spin_unlock_irqrestore(&ep->lock, flags);

	/* 
	 * 如果有就绪的事件发生，那么就调用ep_send_events()将就绪的事件存放到用户态
	 * 内存中，然后返回到用户态，否则判断是否超时，如果没有超时就继续等待就绪事
	 * 件发生，如果超时就返回用户态。
	 */
	if (!res && eavail &&
	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
		goto fetch_events;

	return res;
}
```
　　从ep_poll()函数的实现可以看到，如果有就绪事件发生，则调用ep_send_events()函数做进一步处理，在ep_send_events()函数中又会调用ep_scan_ready_list()函数获取epoll对象eventpoll中的rdllist链表。由于在我们扫描处理eventpoll中的rdllist链表的时候可能同时会有就绪事件发生，这个时候为了保证数据的一致性，在这个时间段内发生的就绪事件会临时存放在eventpoll对象的ovflist链表成员中，待rdllist处理完毕之后，再将ovflist中的内容移动到rdllist链表中，等待下次epoll_wait()的调用。
　　当ep_scan_ready_list()函数获取到rdllist链表中的内容之后，会调用ep_send_events_proc()进行扫描处理，即遍历rdllist链表中的epitem对象，针对每一个epitem对象调用ep_item_poll()函数去获取就绪事件的掩码，如果掩码不为0，说明该epitem对象对应的事件发生了，那么就将其对应的struct epoll_event类型的对象拷贝到用户态指定的内存中；如果掩码为0，则直接处理下一个epitem。注意此时在调用ep_item_poll()函数的时候没有设置poll_table的_qproc回调函数成员，所以只会尝试去获取就绪事件的掩码，该函数的实现如下：
```c
static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
			       void *priv)
{
	struct ep_send_events_data *esed = priv;
	int eventcnt;
	unsigned int revents;
	struct epitem *epi;
	struct epoll_event __user *uevent;
	struct wakeup_source *ws;
	poll_table pt;

	init_poll_funcptr(&pt, NULL);

	/* 
	 * 这里会循环遍历就绪事件对应的item对象组成的链表，依次将链表中item对象
	 * 对应的就绪事件拷贝到用户态，最多拷贝用户态程序指定的就绪事件数目。
	 */
	for (eventcnt = 0, uevent = esed->events;
	     !list_empty(head) && eventcnt < esed->maxevents;) {
		epi = list_first_entry(head, struct epitem, rdllink);
		ws = ep_wakeup_source(epi);
		if (ws) {
			if (ws->active)
				__pm_stay_awake(ep->ws);
			__pm_relax(ws);
		}

		list_del_init(&epi->rdllink);

		/* 
		 * 虽然这里也会调用ep_item_poll()，但是pt->_qproc这个回调函数并没有设置，
		 * 这种情况下文件对象的poll回调函数就只会去获取就绪事件对应的掩码值，
		 * 因为当pt->_qproc会空时，poll回调函数中调用的poll_wait()什么事情都不做
		 * 就返回，所以poll回调函数就只会去获取事件掩码值。
		 */
		revents = ep_item_poll(epi, &pt);

		/* 如果revents不为0，说明确实有就绪事件发生，那么就将就绪事件拷贝到用户态内存中 */
		if (revents) {
			/* 将就绪事件及其data拷贝到用户态内存中 */
			if (__put_user(revents, &uevent->events) ||
			    __put_user(epi->event.data, &uevent->data)) {
				list_add(&epi->rdllink, head);
				ep_pm_stay_awake(epi);
				return eventcnt ? eventcnt : -EFAULT;
			}
			eventcnt++;
			uevent++;
			if (epi->event.events & EPOLLONESHOT)
				epi->event.events &= EP_PRIVATE_BITS;
			/*
			 * 这个地方就是epoll中特有的EPOLLET边缘触发逻辑的实现，即当一个             * 就绪事件拷贝到用户态内存后判断这个事件类型是否包含了EPOLLET位，
			 * 如果没有，则将该事件对应的epitem对象重新加入到epoll的rdllist链
			 * 表中，用户态程序下次调用epoll_wait()返回时就又能获取该epitem了
			 */
			else if (!(epi->event.events & EPOLLET)) {
				/* 将水平触发类型的就绪事件重新加入到ep->rdllist链表中 */
				list_add_tail(&epi->rdllink, &ep->rdllist);
				ep_pm_stay_awake(epi);
			}
		}
	}

	return eventcnt;
}
```
　　我们知道，epoll还有另外一个重要特性就是其支持边缘触发（ET）和水平触发（LT）两种工作模式；以网络套接字为例，对于边缘触发的工作模式，当一个新的事件到来的时候，应用程序可以通过epoll_wait()系统调用获取到这个就绪事件，但是如果用户态应用程序没有一次性处理完这个事件对应的套接字缓冲区的话，那么在这个套接字没有新事件到来之前，epoll_wait()都不会在返回这个事件了；而在水平触发的工作模式下，只要某个事件对应的套接字缓冲区中还有数据没有处理完，那么在调用epoll_wait()的时候总能获取到这个就绪事件，那么在epoll中是如何实现水平触发和边缘触发这两种工作模式的呢？epoll中的实现方式十分简洁，就是在ep_send_events_proc()函数扫描rdllist链表的时候，对于每一个有就绪事件发生的epitem对象，epoll都会判断该epitem对象中存放的用户态传递进来的事件掩码是否包含了EPOLLET位，如果没有包含这个EPOLLET位，那么epoll就会将epitem对象再次加入到rdllist链表中。这样用户态应用程序再次调用epoll_wait()的时候，就又可以在rdllist链表中获取到这个epitem对象了，如果在两次调用epoll_wait()的时间内，用户态应用程序没有处理完这个事件对应的套接字缓冲区中的内容，那么后面那次调用epoll_wait()的时候，就又可以通过调用ep_item_poll()函数获取到epitem对象对应的就绪事件了，这就是水平触发工作模式的原理。
**参考：**
1、	[EPOLL Linux内核源代码实现原理分析](http://blog.chinaunix.net/uid-26339466-id-3292595.html)

** 本文作者： TitenWang **
** 本文链接： https://titenwang.github.io/2017/10/05/implementation-of-epoll/ **
** 版权声明： 本博客所有文章除特别声明外，均采用[ CC BY-NC-ND 3.0 ](https://creativecommons.org/licenses/by-nc-nd/3.0/cn/)许可协议。**