---
title: Linux中rps/rfs的原理及实现
date: 2017-07-09 16:25:50
tags:
  - eBPF
  - 网络编程
  - 协议栈
categories: eBPF
type: "categories"
---

　　rps的全称是Receive Package Steering，rfs的全称是Receive Flow Steering，rps和rfs是google的工程师提供的两个补丁，用以在软件层面实现报文在多个cpu之间的负载均衡以及提高报文处理的缓存命中率。rps和rfs出现的原因主要有以下两个：
　　1、	对于多队列网卡，网卡硬件接收队列与cpu核数在数量上不匹配导致报文在cpu之间分配不均。
　　2、	对于单队列网卡，rps和rfs可以在软件层面将报文平均分配到多个cpu上。<!-- more -->
　　关于rps和rfs更为详细的介绍，可以参考LWN上面的文章，链接于文档末尾参考部分给出。下面分别介绍rps和rfs的原理及其具体实现。
## rps
　　rps，即Receive Package Steering，其原理是单纯地以软件方式实现接收的报文在cpu之间平均分配，即利用报文的hash值找到匹配的cpu，然后将报文送至该cpu对应的backlog队列中进行下一步的处理。上面提到的报文hash值，可以是由网卡计算得到，也可以是由软件计算得到，具体的计算也因报文协议不同而有所差异，以tcp报文为例，tcp报文的hash值是根据四元组信息，即源ip、源端口、目的ip和目的端口进行hash计算得到的。
### rps中的cpu位图
　　上面提到，rps是利用报文的hash值找到对应的cpu，然后将报文送至该cpu的backlog队列来实现报文在多个cpu之间的负载均衡的。所以首先需要知道哪些cpu核会参与报文的分发处理。Linux是通过配置文件的方式指定哪些cpu核参与到报文的分发处理，配置文件存放的路径是：/sys/class/net/(dev)/queues/rx-(n)/rps_cpus，我们以单网卡单队列、8核cpu为例，cpu核号从0开始算起，假设我们用第7个核来处理网卡中断，我们执行如下命令设置第0、2、4、6个cpu核参与到网卡的分发处理中来：
```shell
# echo 55 > /sys/class/net/eth0/queues/rx-0/rps_cpus
```
　　当我们设置好该配置文件之后，内核就会去获取该配置文件的内容，然后根据解析的结果生成一个用于参与报文分发处理的cpu列表（实际实现是一个柔性数组），这样当收到报文之后，就可以建立起hash-cpu的映射关系了。
　　在看解析函数的实现之前，需要先对相应的数据结构有所了解。从上面的配置文件中我们可以看到配置文件是放置在网卡的接口队列下的，其实对应的内核实现中也是将解析的结果放置在网卡的接收队列实例中的，其定义如下：
```c
struct netdev_rx_queue {
#ifdef CONFIG_RPS
	struct rps_map __rcu		*rps_map;
	……
#endif
	……
} ____cacheline_aligned_in_smp;
```
　　用来存放解析结果的就是网卡硬件接收队列实例的rps_map成员，其类型定义如下：
```c
struct rps_map {
	unsigned int len;  // cpus柔性数组的长度
	struct rcu_head rcu;
	u16 cpus[0];  // 存放cpu id的柔性数组
};
```
　　struct rps_map的cpus成员是用来记录配置文件中配置的参与报文分发处理的cpu核号的柔性数组，而len成员就是cpus柔性数组的长度。
　　了解存放配置文件解析结果的数据结构之后，再看对应的解析函数就比较容易理解了，其实现如下：
```c
static ssize_t store_rps_map(struct netdev_rx_queue *queue,
		      struct rx_queue_attribute *attribute,
		      const char *buf, size_t len)
{
	struct rps_map *old_map, *map;
	cpumask_var_t mask;
	int err, cpu, i;
	static DEFINE_MUTEX(rps_map_mutex);
	……
	if (!alloc_cpumask_var(&mask, GFP_KERNEL))
		return -ENOMEM;

	/* 解析缓冲区buf中的cpu位图 */
	err = bitmap_parse(buf, len, cpumask_bits(mask), nr_cpumask_bits);
	if (err) {
		free_cpumask_var(mask);
		return err;
	}

	/* 申请map对象 */
	map = kzalloc(max_t(unsigned int,
	    RPS_MAP_SIZE(cpumask_weight(mask)), L1_CACHE_BYTES),
	    GFP_KERNEL);
	……

	/* 根据mask对象中的cpu掩码获取对应的cpu id，并存放到map->cpus数组中 */
	i = 0;
	for_each_cpu_and(cpu, mask, cpu_online_mask)
		map->cpus[i++] = cpu;

	/* 记录数组的长度 */
	if (i)
		map->len = i;
	else {
		kfree(map);
		map = NULL;
	}
	……
	if (map)
		static_key_slow_inc(&rps_needed);
	……
	free_cpumask_var(mask);
	return len;
}
```
　　根据上面设定的rps_cpus配置文件内容，以及这里的解析函数实现，我们可以知道在解析完成之后，struct rps_map类型的实例中len的值为4，而cpus数组的内容则是{0,2,4,6}。
### rps中的负载均衡
　　从配置文件中获取到了参与报文分发处理的cpu列表信息之后，就可以根据报文的hash值，从cpu列表选取一个cpu进行后续的报文处理了。从cpu列表中获取核号的实现如下：
```c
static int get_rps_cpu(struct net_device *dev, struct sk_buff *skb,
		       struct rps_dev_flow **rflowp)
{
	const struct rps_sock_flow_table *sock_flow_table;
	struct netdev_rx_queue *rxqueue = dev->_rx;
	struct rps_dev_flow_table *flow_table;
	struct rps_map *map;
	int cpu = -1;
	u32 tcpu;
	u32 hash;
	……
	/*
	 * 如果程序进入这个流程，说明使用rps而不是rfs来获取处理该报文所在流的cpu。
	 * 与rfs不同，rps是单纯用报文的hash值来分发报文，而不关注处理该流中报文的
	 * 应用程序所在的cpu。map->cpus数组在函数store_rps_map()中已经初始化，所以
	 * 这里就直接用报文中存放的hash值去索引map->cpus数组即可。
	 */
	if (map) {
		tcpu = map->cpus[reciprocal_scale(hash, map->len)];
		if (cpu_online(tcpu)) {
			cpu = tcpu;
			goto done;
		}
	}

done:
	return cpu;
}
```
　　对于函数get_rps_cpu()，这里只选取了和rps处理相关的部分，rfs的部分在介绍rfs实现的时候在详细给出。从该函数中rps处理部分可以看到获取用于分发处理报文的cpu时，是用报文对应的hash值与上cpu列表掩码之后的得到的值当作cpu列表的索引值，进而获取到对应的cpu核号。
### rps的报文处理流程
　　在了解了rps如何根据报文的hash值来选取分发处理该报文的cpu之后，我们自然而然会想到应该在什么时候获取到这个cpu呢，也就是何时调用get_rps_cpu()函数呢？我们知道，Linux在处理网卡收包中断的时候分为了上半部处理和下半部处理，对于支持NAPI接口的驱动而言，在上半部的处理主要就是将设备加入到cpu的私有数据softnet_data的待轮询设备列表中，而下半部主要就是调用poll回调函数从网卡缓冲区中获取报文，然后上送给协议栈。而函数get_rps_cpu()就是在下半部获取到报文信息，构造好skb对象，调用netif_receive_skb()之后，准备调用__netif_receive_skb()之前被调用的，也就是在netif_receive_skb_internal()函数中被调用的，其实现如下：
```c
static int netif_receive_skb_internal(struct sk_buff *skb)
{
	int ret;

	……
	rcu_read_lock();

#ifdef CONFIG_RPS
	if (static_key_false(&rps_needed)) {
		struct rps_dev_flow voidflow, *rflow = &voidflow;
		/* 获取处理该报文的cpu以及对应的rps_dev_flow流表 */
		int cpu = get_rps_cpu(skb->dev, skb, &rflow);

		/* 
		 * 将报文放入到该cpu私有数据对象softnet_data中的input_pkt_queue中，那
		 * 什么时候去input_pkt_queue队列中去取呢?在cpu的私有数据对象softnet_data
		 * 中，有一个backlog的成员，该成员的类型为struct napi_struct，在函数
		 * enqueue_to_backlog()中，会将backlog加入到cpu的待轮询设备列表中，并触发
		 * 软中断，在软中断处理函数net_rx_action()中会依次遍历待轮询设备列表中的
		 * 设备，并调用设备注册的poll回调函数来进行报文处理。而对应于rps，其poll
		 * 回调函数为process_backlog()，该函数会调用__netif_receive_skb()函数将
		 * 报文送至协议栈进行处理。
		 */
		if (cpu >= 0) {
			ret = enqueue_to_backlog(skb, cpu, &rflow->last_qtail);
			rcu_read_unlock();
			return ret;
		}
	}
#endif
	/* 如果没有rps/rfs的话，报文直接就送上协议栈了。 */
	ret = __netif_receive_skb(skb);
	rcu_read_unlock();
	return ret;
}
```
　　从函数netif_receive_skb_internal()的实现中可以看到，当rps调用函数get_rps_cpu()获取到用于分发处理报文的目标cpu之后，如果目标cpu有效，则会调用enqueue_to_backlog()函数做进一步处理，而不是直接调用函数__netif_receive_skb()将报文送上协议栈处理。那函数enqueue_to_backlog()做了什么工作呢，而进入了rps处理流程的报文又是什么时候会被上送到协议栈处理呢？
　　先来看下函数enqueue_to_backlog()主要做了哪些工作，其实该函数做的主要事情是尝试将报文加入到cpu私有数据对象softnet_data的input_pkt_queue队列中，如果该队列满了就将设备加入到cpu私有数据对象softnet_data的待轮询设备列表中，并触发一个收包软中断(软中断处理函数会调用poll回调函数从该队列中取数据)，然后再尝试将报文加入到input_pkt_queue队列中。该函数的实现如下：
```c
static int enqueue_to_backlog(struct sk_buff *skb, int cpu,
			      unsigned int *qtail)
{
	struct softnet_data *sd;
	unsigned long flags;
	unsigned int qlen;

	sd = &per_cpu(softnet_data, cpu);
	local_irq_save(flags);
	rps_lock(sd);
	……
	qlen = skb_queue_len(&sd->input_pkt_queue);
	if (qlen <= netdev_max_backlog && !skb_flow_limit(skb, qlen)) {
		if (qlen) {
enqueue:
			/* 将报文添加到sd->input_pkt_queue队列的末尾，并更新队列尾部索引 */
			__skb_queue_tail(&sd->input_pkt_queue, skb);
			input_queue_tail_incr_save(sd, qtail);
			rps_unlock(sd);
			local_irq_restore(flags);
			return NET_RX_SUCCESS;
		}

		/* 
		 * 程序执行到这里表明sd->input_pkt_queue已经满了，这个时候需要将sd->backlog加入到
		 * cpu私有数据的待轮询设备列表sd->poll_list中，并触发接收软中断NET_RX_SOFTIRQ，
		 * 软中断处理函数net_rx_action()中会从调用poll回调函数从input_pkt_queue队列中取包。
		 * 这样input_pkt_queue队列中又有空闲位置用来存放报文了。
		 * rps/rfs注册的poll回调的实现具体可以参考process_backlog()。
		 */
		if (!__test_and_set_bit(NAPI_STATE_SCHED, &sd->backlog.state)) {
			if (!rps_ipi_queued(sd))
				____napi_schedule(sd, &sd->backlog);
		}
		goto enqueue;
	}

drop:
	rps_unlock(sd);
	local_irq_restore(flags);
	……
	return NET_RX_DROP;
}
```
　　在函数enqueue_to_backlog()中我们看到，如果cpu私有数据对象softnet_data的input_pkt_queue队列满了，那么就会将设备对象backlog(其实是struct napi_struct类型的实例)加入到cpu私有数据对象softnet_data的待轮询设备列表中，并触发一个NET_RX_SOFTIRQ的收包软中断。当中断子系统检测到有软中断发生，就会调用软中断的处理函数net_rx_action()，在函数net_rx_action()中则会遍历cpu私有数据对象softnet_data的待轮询的设备列表，当遍历到rps对应的backlog设备对象的时候，就会调用该设备对象注册的poll回调函数，即process_backlog()来处理input_pkt_queue队列。函数process_backlog()的实现如下：
```c
static int process_backlog(struct napi_struct *napi, int quota)
{
	struct softnet_data *sd = container_of(napi, struct softnet_data, backlog);
	bool again = true;
	int work = 0;
	……
	napi->weight = weight_p;
	while (again) {
		struct sk_buff *skb;

		/* 循环获取sd->process_queue队列中的报文，然后调用__netif_receive_skb()送至协议栈 */
		while ((skb = __skb_dequeue(&sd->process_queue))) {
			rcu_read_lock();
			__netif_receive_skb(skb);
			rcu_read_unlock();
			input_queue_head_incr(sd);
			if (++work >= quota)
				return work;
		}

		local_irq_disable();
		rps_lock(sd);
		if (skb_queue_empty(&sd->input_pkt_queue)) {
			napi->state = 0;
			again = false;
		} else {
			/* 
			 * 将sd->input_pkt_queue队列内容拼接到sd->process_queue队列中，从
			 * 上面的循环可以看出，process_backlog()并不是直接处理sd->input_pkt_queue队列的，
			 * 而是通过sd->process_queue队列做中转，处理完process_queue队列报文之后，再将
			 * input_pkt_queue队列的内容拼接到process_queue队列中，下次又继续处理process_queue队列。
			 */
			skb_queue_splice_tail_init(&sd->input_pkt_queue,
						   &sd->process_queue);
		}
		rps_unlock(sd);
		local_irq_enable();
	}

	return work;
}
```
　　从函数process_backlog()的实现我们可以看到处理input_pkt_queue队列其实就是从该队列中取出报文，然后调用__netif_receive_skb()上报文送至协议栈进行后续处理。所以对于本节开始提出的问题—rps处理流程中的报文是何时被送至协议栈的，现在就比较清晰了，对于没有rps的处理流程，如果采用的是NAPI收包方式，那么软中断处理函数net_rx_action()会调用网卡驱动注册的poll回调函数从网卡中获取到报文数据后就将报文数据上送至协议栈；而对于有rps的处理流程，软中断处理函数net_rx_action()会调用网卡驱动注册的poll回调函数从网卡中获取到报文数据后，暂时不直接送至协议栈，而是选择一个目标cpu，将报文放到该cpu私有数据对象softnet_data的input_pkt_queue队列中，待对列input_pkt_queue满了之后，就将该cpu对应的backlog设备对象加入到该cpu的待轮询设备列表中，并触发软中断，软中断处理函数轮询到backlog设备对象后，调用poll回调函数process_backlog()从input_pkt_queue队列中取出报文，再上送至协议栈。
　　到这里rps的原理及实现就介绍的差不多了。下面介绍下rfs的原理及实现。
## rfs
　　从rps的原理来看，我们知道rps只是根据报文的hash值从分发处理报文的cpu列表中选取一个目标cpu，这样虽然负载均衡的效果很好，但是当用户态处理报文的cpu和内核处理报文软中断的cpu不同的时候，就会导致cpu的缓存不命中，影响性能。而rfs就是用来处理这种情况的，rfs的目标是通过指派处理报文的应用程序所在的cpu来在内核态处理报文，以此来增加cpu的缓存命中率。所以rfs相比于rps，主要差别就是在选取分发处理报文的目标cpu上，而rfs还需要依靠rps提供的机制进行报文的后续处理。
　　rfs实现指派处理报文的应用程序所在的cpu来在内核态处理报文这一目标主要是依靠两个流表来实现的，其中一个是设备流表，记录的是上次在内核态处理该流中报文的cpu；另外一个是全局的socket流表，记录的是流中的报文渴望被处理的目标cpu。
### rfs的设备流表
　　设备流表定义在网卡的硬件接收队列中，如下：
```c
struct netdev_rx_queue {
#ifdef CONFIG_RPS
	……
	struct rps_dev_flow_table __rcu	*rps_flow_table;
#endif
	……
} ____cacheline_aligned_in_smp;
```
　　设备流表的类型为struct rps_dev_flow_table，其定义为：
```c
struct rps_dev_flow_table {
	unsigned int mask;
	struct rcu_head rcu;
	struct rps_dev_flow flows[0];
};
```
　　设备流表中主要包括一个类型为struct rps_dev_flow的柔性数组，以及存放着柔性数组大小的成员mask。而struct rps_dev_flow类型的实例则主要包括存放着上次处理该流中报文的cpu以及所在cpu私有数据对象softnet_data的input_pkt_queue队列尾部索引的两个成员，其定义如下：
```c
struct rps_dev_flow {
	u16 cpu;  /* 处理该流的cpu */
	u16 filter;
	unsigned int last_qtail;  /* sd->input_pkt_queue队列的尾部索引，即该队列长度 */
};
```
　　上面说到设备流表中的mask成员是类型为struct rps_dev_flow的柔性数组的大小，也就是流表项的数目，这个主要是通过配置文件 /sys/class/net/(dev)/queues/rx-(n)/rps_flow_cnt进行指定的。当我们设置了配置文件，那么内核就会获取到数据，并初始化网卡硬件接收队列中的设备流表成员rps_flow_table，初始化过程可以参考函数store_rps_dev_flow_table_cnt()。
### rfs的全局socket流表
　　rps_sock_flow_table是一个全局的数据流表，这个表中包含了数据流渴望被处理的CPU。这个CPU是当前处理流中报文的应用程序所在的CPU。全局socket流表会在调recvmsg，sendmsg (特别是inet_accept(), inet_recvmsg(), inet_sendmsg(), inet_sendpage() and tcp_splice_read())，被设置或者更新。
全局socket流表rps_sock_flow_table的定义如下：
```c
struct rps_sock_flow_table {
	u32	mask;
	u32	ents[0] ____cacheline_aligned_in_smp;
};
```
　　struct rps_sock_flow_table类型的mask成员存放的就是ents这个柔性数组的大小，该值也是通过配置文件的方式指定的，相关的配置文件为 /proc/sys/net/core/rps_sock_flow_entries。
　　上面说到全局socket流表会在调用recvmsg()等函数时被更新，而在这些函数中是通过调用函数sock_rps_record_flow()来更新或者记录流表项信息的，而sock_rps_record_flow()中最终又是调用函数rps_record_sock_flow()来更新ents柔性数组的，该函数实现如下：
```c
static inline void rps_record_sock_flow(struct rps_sock_flow_table *table,
					u32 hash)
{
	if (table && hash) {
		/* 用hash值对表的掩码求余，获取对应的表项索引 */
		unsigned int index = hash & table->mask;

		/* 保留hash值的高位，将用来存放cpu id的低位清零 */
		u32 val = hash & ~rps_cpu_mask;

		/* We only give a hint, preemption can change CPU under us */
		/* 
		 * raw_smp_processor_id()获取当前cpu，经过上面和这两部分，val存放了
		 * 高位hash值和cpu id，为什么在高位还需要保留部分hash值而不直接存放cpu
		 * id呢?原因是因为在get_rps_cpu()函数中，还会用val中存放的高位hash值
		 * 来校验skb中的hash值是否在rps_sock_flow_table中有对应的记录。详情可以
		 * 参考get_rps_cpu()。
		 */
		val |= raw_smp_processor_id();

		/* 记录hash值对应的cpu */
		if (table->ents[index] != val)
			table->ents[index] = val;
	}
}
```
　　在该函数中，table传递的就是全局socket流表rps_sock_flow_table的地址，hash则是流hash，也就是通过报文计算得到的hash值。从这个函数的实现我们可以看到，ents柔性数组中存放的不仅仅是当前处理流中报文的应用程序所在的cpu，其中还包括了流的hash值的高位数据，其内存布局如下：
<div align="center"> {% asset_img ents项内存布局.png 图1 ents项内存布局 %} </div>
	
　　为什么流表项中不直接存放cpu信息，还要存放hash值的高位部分呢？这个主要使用来做校验的，这部分可以在函数get_rps_cpu()中看到。介绍了rfs负载均衡策略中会使用到的两种流表之后，再来理解其负载均衡策略就比较清晰了。
### rfs的负载均衡
　　上面说到，rfs相比于rps，主要差别就是在选取分发处理报文的目标cpu上。所以rfs负载均衡策略的实现也主要体现在函数get_rps_cpu()中，其实现如下：
```c
static int get_rps_cpu(struct net_device *dev, struct sk_buff *skb,
		       struct rps_dev_flow **rflowp)
{
	const struct rps_sock_flow_table *sock_flow_table;
	struct netdev_rx_queue *rxqueue = dev->_rx;
	struct rps_dev_flow_table *flow_table;
	struct rps_map *map;
	int cpu = -1;
	u32 tcpu;
	u32 hash;
	……
	/* 从接收队列中获取记录着上次处理报文所在流的cpu的流表rps_dev_flow_table */
	flow_table = rcu_dereference(rxqueue->rps_flow_table);
	map = rcu_dereference(rxqueue->rps_map);
	if (!flow_table && !map)
		goto done;

	skb_reset_network_header(skb);
	/* 
	 * 获取报文的hash值，如果硬件计算了，就用硬件计算的，否则进行软件计算，
	 * 后续会用这个hash值去索引rps_dev_flow_table和rps_sock_flow_table两个流表
	 */
	hash = skb_get_hash(skb);
	if (!hash)
		goto done;

	sock_flow_table = rcu_dereference(rps_sock_flow_table);
	if (flow_table && sock_flow_table) {
		struct rps_dev_flow *rflow;
		u32 next_cpu;
		u32 ident;

		/* First check into global flow table if there is a match */
		ident = sock_flow_table->ents[hash & sock_flow_table->mask];
		/* 
		 * 从函数rps_record_sock_flow()中可以看到，ident由两部分组成，即
		 * 高位是hash值的一部分，低位是cpu id。所以这里用异或操作符(^)来判断
		 * ident高位部分和hash值是否一样，然后低位又和cpu id掩码按位取反的值
		 * 进行按位与操作，经过这样的计算，如果最后面表达式值为0，说明hash值
		 * 在rps_sock_flow_table中有记录，则用rfs进行后续处理，否则用rps。
		 */
		if ((ident ^ hash) & ~rps_cpu_mask)
			goto try_rps;

		/* 
		 * 在函数rps_record_sock_flow()中我们可以看到，rps_sock_flow_table全局
		 * 流表中ents数组中存放的并不是单纯的cpu id，而是hash值高位和cpu id的组合。
		 * 所以这里用cpu掩码来获取cpu id。全局流表rps_sock_flow_table中存放的是流
		 * 渴望被处理的cpu。
		 */
		next_cpu = ident & rps_cpu_mask;

		/* tcpu是上次处理这个流的cpu */
		rflow = &flow_table->flows[hash & flow_table->mask];
		tcpu = rflow->cpu;

		/* 
		 * 如果流渴望被处理的cpu和上次处理这个流的cpu不同，说明处理流的应用程序
		 * 被调度到了另外一个核next_cpu上，所以这里需要将tcpu设置为流渴望被
		 * 处理的cpu，这个流所在报文需要被送到next_cpu上。
		 * 对于最后一个条件，结合tcpu!=next_cpu，可以这样理解:处理流的应用程序
		 * 目前被调度到了另外一个cpu上，而上次处理该流中报文的cpu(即tcpu)中又没有
		 * 剩余没有处理的这条流中的报文，那么就可以将上次处理该流报文的cpu更新为
		 * 应用程序所在的cpu；如果tcpu中还有剩余的属于该流的报文没有处理完，那么
		 * 就会继续将本次处理的属于该流的新报文继续让tcpu来处理，这样可以保证同
		 * 属于一条流的报文可以被有序的处理，而不会乱序。
		 */
		if (unlikely(tcpu != next_cpu) &&
		    (tcpu >= nr_cpu_ids || !cpu_online(tcpu) ||
		     ((int)(per_cpu(softnet_data, tcpu).input_queue_head -
		      rflow->last_qtail)) >= 0)) {
			tcpu = next_cpu;
			rflow = set_rps_cpu(dev, skb, rflow, next_cpu);
		}

		if (tcpu < nr_cpu_ids && cpu_online(tcpu)) {
			*rflowp = rflow;
			cpu = tcpu;
			goto done;
		}
	}
	……
done:
	return cpu;
}
```
　　从其实现可以看到，rfs的负载均衡策略主要是通过判断报文的hash值(亦即流hash值)所对应的两个流表（设备流表和全局socket流表）中的记录的cpu是否相同，如果相同，那么该cpu就会被当作内核态处理该流中报文的cpu；如果不相同，那么在满足一定条件的情况需要将内核态处理该流中报文的cpu设置成用户态处理该流中报文的应用程序所在的cpu，如果条件不满足的话，那么还是会继续沿用上一次在内核态处理该流中的报文的cpu来进行本次报文的处理。切换用于在内核态处理流中报文的cpu所需要满足的任一条件如下：
　　1、	上一次在内核态处理该流中报文的cpu是离线的。
　　2、	上一次在内核态处理该流中报文的cpu是无效值（初始化会设置为nr_cpu_ids，该值无效）。
　　3、上一次在内核态处理该流中报文的cpu的私有数据对象softnet_data的input_pkt_queue队列的头部索引input_queue_head的值大于设备流表中记录的input_pkt_queue队列的尾部索引值，这说明上一次在内核态处理该流中报文的cpu的input_pkt_queue队列中已经没有属于该流的未上送给协议栈的报文了，此时如果切换在内核态处理属于该流的报文的cpu也不会导致属于该流的报文被乱序处理了。
　　到这里rfs的原理和实现就介绍完了。



**参考：**
1、	[rps：Receive Package Steering](https://lwn.net/Articles/362339/)
2、	[rfs：Receive Flow Steering](https://lwn.net/Articles/382428/)

** 本文作者： TitenWang **
** 本文链接： https://titenwang.github.io/2017/07/09/implementation-of-rps-and-rfs/ **
** 版权声明： 本博客所有文章除特别声明外，均采用[ CC BY-NC-ND 3.0 ](https://creativecommons.org/licenses/by-nc-nd/3.0/cn/)许可协议。**
