---
title: DPDK中的无锁环形缓冲区
date: 2016-11-20 15:55:12
tags: 
  - DPDK
  - 网络编程
categories: DPDK
type: "categories"
---

　　DPDK中实现了无锁环形缓冲区，支持单生产者或者多生产者入队列，单消费者或多消费者出队列。<!-- more -->下面会记录dpdk中是如何管理所有使用的无锁环形缓冲区以及无锁环形缓冲区中支持的一些操作。
## 无锁环形缓冲区的组织
　　DPDK中定义的无锁环形缓冲区对象如下：
```c
struct rte_ring {
	char name[RTE_MEMZONE_NAMESIZE];    /**< Name of the ring. */
	int flags;                 /**< Flags supplied at creation. */
	const struct rte_memzone *memzone;
			/**< Memzone, if any, containing the rte_ring */

	/** Ring producer status. */
	struct prod {
		uint32_t watermark;      /**< Maximum items before EDQUOT. */
		uint32_t sp_enqueue;     /**< True, if single producer. */
		uint32_t size;           /**< Size of ring. */
		uint32_t mask;           /**< Mask (size-1) of ring. */
		volatile uint32_t head;  /**< Producer head. */
		volatile uint32_t tail;  /**< Producer tail. */
	} prod __rte_cache_aligned;

	/** Ring consumer status. */
	struct cons {
		uint32_t sc_dequeue;     /**< True, if single consumer. */
		uint32_t size;           /**< Size of the ring. */
		uint32_t mask;           /**< Mask (size-1) of ring. */
		volatile uint32_t head;  /**< Consumer head. */
		volatile uint32_t tail;  /**< Consumer tail. */
#ifdef RTE_RING_SPLIT_PROD_CONS
	} cons __rte_cache_aligned;
#else
	} cons;
#endif

#ifdef RTE_LIBRTE_RING_DEBUG
	struct rte_ring_debug_stats stats[RTE_MAX_LCORE];
#endif

	void * ring[0] __rte_cache_aligned; 
	                        /**< Memory space of ring starts here.
	                         * not volatile so need to be careful
	                         * about compiler re-ordering */
};
```
　　从上面的定义可以看出，无锁环形缓冲区对象中定义了一个生产者对象和一个消费者对象，对应的也就是缓冲区的写对象和读对象。另外，也可以看出无锁环形缓冲区在内存中的组织形式是前面是无锁环形缓冲区对象本身，然后紧接着就是实际用于存储内容的环形队列，在某一时刻其内存布局如下图所示：
 <div align="center"> {% asset_img 无锁环形缓冲区内存布局.png 图1 无锁环形缓冲区内存布局 %} </div>
　　无锁环形缓冲区是一种通用的数据结构，所以可能会在多个地方使用，在dpdk中就会有多种情况会使用，比如内存池。所以随之而来的一个疑问就是dpdk是如何管理其所使用的所有的无锁环形缓冲区的呢？从它的源码实现(rte_ring.c/rte_ring.h)中我们可以找到答案，与此相关的部分源码如下：
```c
TAILQ_HEAD(rte_ring_list, rte_tailq_entry);

static struct rte_tailq_elem rte_ring_tailq = {
	.name = RTE_TAILQ_RING_NAME,
};
EAL_REGISTER_TAILQ(rte_ring_tailq)

struct rte_ring *
rte_ring_create(const char *name, unsigned count, int socket_id,
		unsigned flags)
{
	char mz_name[RTE_MEMZONE_NAMESIZE];
	struct rte_ring *r;
	struct rte_tailq_entry *te;
	const struct rte_memzone *mz;
	ssize_t ring_size;
	int mz_flags = 0;
	struct rte_ring_list* ring_list = NULL;
	int ret;

	ring_list = RTE_TAILQ_CAST(rte_ring_tailq.head, rte_ring_list);

	/* get the size of memory occupied by ring */
	ring_size = rte_ring_get_memsize(count);
	……
	te = rte_zmalloc("RING_TAILQ_ENTRY", sizeof(*te), 0);
	if (te == NULL) {
		RTE_LOG(ERR, RING, "Cannot reserve memory for tailq\n");
		rte_errno = ENOMEM;
		return NULL;
	}

	rte_rwlock_write_lock(RTE_EAL_TAILQ_RWLOCK);
	mz = rte_memzone_reserve(mz_name, ring_size, socket_id, mz_flags);
	if (mz != NULL) {
		r = mz->addr;
		/* no need to check return value here, we already checked 
		 * the arguments above */
		rte_ring_init(r, name, count, flags);

		/* 低维尾队列的entry存放着rte_ring的管理对象地址 */
		te->data = (void *) r;
		r->memzone = mz;

		/* 将存放着环形缓冲区对象的尾队列entry插入到低维尾队列的末端 */
		TAILQ_INSERT_TAIL(ring_list, te, next);
	} else {
		r = NULL;
		RTE_LOG(ERR, RING, "Cannot reserve memory\n");
		rte_free(te);
	}
	rte_rwlock_write_unlock(RTE_EAL_TAILQ_RWLOCK);

	return r;
}
```
　　从上面的源码我们可以知道，dpdk是用尾队列来管理其所使用的所有无锁环形缓冲区的，也就是说一个尾队列中的元素就是一个无锁环形缓冲区对象。那用来管理所有无锁环形缓冲区的尾队列，dpdk又是如何管理的呢？在函数rte_ring_create()中可以看到管理着无锁环形缓冲区的尾队列头部是存放在一个类型为struct rte_tailq_elem的全局变量rte_ring_tailq的head成员中，其中struct rte_tailq_elem定义如下：
```c
struct rte_tailq_elem {
	struct rte_tailq_head *head;
	TAILQ_ENTRY(rte_tailq_elem) next;
	const char name[RTE_TAILQ_NAMESIZE];
};
```
　　从struct rte_tailq_elem的定义可以看到，管理着无锁环形缓冲区尾队列的头部是另外一个尾队列的一个元素，类似的管理方式还用在了dpdk的内存池等数据结构中，所以在dpdk中采用了两级尾队列来管理所使用的数据结构。其实这样说也不完整，因为除了用二级尾队列来管理所使用的数据结构之外，dpdk还用了一个全局共享内存中的列表数组rte_config.mem_config->tailq_head[RTE_MAX_TAILQ]来分别存储这个二级尾队列中低维尾队列存放的元素，即管理某一种特定数据结构的尾队列头部。
## 无锁环形缓冲区的操作
　　记录完了dpdk中对无锁环形缓冲区的管理方式，接下来对dpdk中实现的无锁环形缓冲区所支持的操作做一个记录。从一开始说过无锁环形缓冲区支持单生产者或者多生产者入队列，单消费者或多消费者出队列等操作。其实从另外一个角度还可以说dpdk中的无锁环形缓冲区中支持两种出入队列的模式，即出入队列元素数目固定模式和尽力而为模式，出入队列元素数目固定模式就是说只有进入队列的元素数目达到指定数目才算操着成功，否则失败；而出入队列元素数目尽力而为模式就是说对于指定的数目，如果当时队列状态并不能满足，则以当时队列状态为准，尽可能满足指定的数目，举个例子，如果参数指定需要入队列3个元素，但队列中只剩下2个空闲空间，那么就将其中2个元素入队列，出队列情况同理。
　　下面以两个核同时往队列各写入一个元素来介绍无锁环形缓冲区支持的多生产者入队列功能，其他的操作方式都可以从这里推演出来。在代码中多生产者入队列相关的函数为__rte_ring_mp_do_enqueue()，下面的流程也是根据这个函数整理出来的。
　　1）	初始状态下生产者的头和尾指向了同一个位置。如下图所示：
<div align="center"> {% asset_img 多生产者入队列初始状态.png 图2 多生产者入队列初始状态 %} </div>
　　2）	第一步，在两个核上，将r->prod.head和r->prod.tail分别拷贝到本地的临时变量prod_head和prod_tail中，然后将本地临时变量prod_next指向队列的下一个空闲位置。检查队列中是否有足够的空间，如果没有，则返回失败（如果是写入多个元素，没有足够剩余空间的话，则需要看指定的模式，如果是尽力而为模式，则尽可能往队列中写入元素；否则返回失败）。如下图所示：
<div align="center"> {% asset_img 多生产者入队列第一步.png 图3 多生产者入队列第一步 %} </div>
　　3）	第二步，使用CAS操作指令将本地变量prod_next的值赋值给r->prod.head，即两者指向同一个位置。CAS操作指令有如下特性：
　　a)	如果r->prod.head不等于本地变量prod_head，则CAS操作失败，代码重新从第一步开始执行。
　　b)	如果r->prod.head等于本地变量prod_head，则CAS操作成功，代码继续往后执行。
在这里我们假定在核1上操作成功，那么对于核2该操作就会失败，核2会从第一步重新执行。如下图所示：
<div align="center"> {% asset_img 多生产者入队列第二步.png 图4 多生产者入队列第二步 %} </div>
　　4）	第三步，核2上的CAS操作成功，核1上成功往环形缓冲区中写入了一个元素（obj4），接着核2也成功往环形缓冲区中写入了一个元素（obj5）。如下图所示：
<div align="center"> {% asset_img 多生产者入队列第三步.png 图5 多生产者入队列第三步 %} </div>
　　5）	第四步，在第三步中两个核都已经成功往环形缓冲区写入了一个元素，现在两个核都需要更新r->prod.tail。这里又有一个条件，就是只有r->prod.tail等于本地变量prod_head的核才能去更新r->prod.tail的值。从图中可以看到，目前只有在核1上才能满足这个条件，核2不满足，因此更新r->prod.tail的操作只在核1上进行，核2需要等待核1完成。如下图所示：
<div align="center"> {% asset_img 多生产者入队列第四步.png 图6 多生产者入队列第四步 %} </div>
　　6）	第五步，在第四步中，一旦核1完成了更新r->prod.tail的操作，那么核2也能满足更新r->prod.tail的条件，核2此时也会去更新r->prod.tail。如下图所示：
<div align="center"> {% asset_img 多生产者入队列第五步.png 图7 多生产者入队列第五步 %} </div>
## 无锁环形缓冲区的回绕
　　DPDK中的无锁环形缓冲区还有另一个特性，那就是充分利用了unsigned类型的回绕特点，这样对于缓冲区中已用空间和剩余空间的计算就得到了极大的简化，也使得生产者头和尾、消费者头和尾的下标值不局限在0和size(ring) – 1之间，只要在0和2^32 - 1范围之内即可。这一点可以参考dpdk的开发者文档，其具体实现也包含在了上节介绍的操作流程当中，感兴趣的可以去看下。

**参考文章:**
　　1、http://dpdk.org/doc/guides/prog_guide/

** 本文作者： TitenWang **
** 本文链接： https://titenwang.github.io/2016/11/20/dpdk-ring-buffer/ **
** 版权声明： 本博客所有文章除特别声明外，均采用[ CC BY-NC-ND 3.0 ](https://creativecommons.org/licenses/by-nc-nd/3.0/cn/)许可协议。**