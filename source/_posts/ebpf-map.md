---
title: eBPF中的map
date: 2017-03-26 20:24:01
tags: 
  - eBPF
  - 网络编程
categories: eBPF
type: "categories"
---

　　map是一个通用的key-value存储结构,可以用来存储任意类型的数据。map在eBPF中扮演这十分重要的角色，这个文档将会介绍eBPF目前支持的map类型，以及某一类型map的实现细节。
　　在eBPF中可以利用map在eBPF程序调用之间保存状态信息，也可以利用map在用户态程序和内核之间共享数据等。<!-- more -->正如文档开头所说，map是通用的key-value存储结构，因为在实现的时候是key和value都是当作二进制对象来存储的，所以可以用来存放任意类型的key和value数据。当然，这需要用户在map创建的时候指定key和value对应的类型大小，这样，内核在创建的时候就能根据key和value的大小来申请合适的内存用以存储数据。
　　内核提供了一个系统调用bpf()，以让用户态程序可以根据使用场景来创建合适的map。这个系统调用会返回一个关联了这个map对象的文件描述符，后续用户态程序可以用这个文件描述符来对相应的map对象进行一些操作，如查询、更新和删除，这部分的接口在tools/lib/bpf/bpf.h中定义了。关于这个bpf()系统调用以及map操作接口的详细信息，可以参考相关资料，其中bpf()系统调用相关的信息可以在man page中找到，而map操作相关的接口可以在tools/lib/bpf/bpf.h中看到具体的实现。
　　eBPF支持多种不同类型的map，所以用户在创建map的时候，需要根据应用场景指定一个合适的类型。目前版本支持的map类型定义在文件include/uapi/linux/bpf.h中，如下：
```c
enum bpf_map_type {
    BPF_MAP_TYPE_UNSPEC,
    BPF_MAP_TYPE_HASH,
    BPF_MAP_TYPE_ARRAY,
    BPF_MAP_TYPE_PROG_ARRAY,
    BPF_MAP_TYPE_PERF_EVENT_ARRAY,
    BPF_MAP_TYPE_PERCPU_HASH,
    BPF_MAP_TYPE_PERCPU_ARRAY,
    BPF_MAP_TYPE_STACK_TRACE,
    BPF_MAP_TYPE_CGROUP_ARRAY,
};
```
　　从类型定义的名字中我们大概可以知道这些map所对应的使用场景，如类型BPF_MAP_TYPE_PROG_ARRAY，通用用来存放一个关联了eBPF程序的文件描述符，这样就可以实现在多个eBPF程序之间跳转；而对于BPF_MAP_TYPE_HASH和BPF_MAP_TYPE_ARRAY而言，他们所存储的内容就没有特殊的用途，可以用来存放任意类型的key和value信息。
　　虽然目前eBPF支持多种map类型，但是从类型名字中可以看出，map类型可以分为hash和array两大类，也就对应了两种不同的实现方式。而每一类中不同map类型的实现又有共同的部分，所以懂得了每一大类map共有部分的实现，就对大类内其他类型的map实现有了整体的了解，然后再结合各自应用场景，就可以比较好地掌握适用于不同场景的map的实现。下面会详细介绍hash这一类map的基础实现，也就是BPF_MAP_TYPE_HASH类型的map在内核中的实现。
　　先来看下内核中map对象的定义：
```c
struct bpf_map {
    atomic_t refcnt;
    enum bpf_map_type map_type;
    u32 key_size;
    u32 value_size;
    u32 max_entries;
    u32 map_flags;
    u32 pages;
    struct user_struct *user;
    const struct bpf_map_ops *ops;
    struct work_struct work;
    atomic_t usercnt;
};
```
　　在这个对象结构体中，key_size和value_size分别用于指定map存储的键值大小，而max_entries则指定这个map对象中可以存储的键值对数量，在实现map内部存储的时候会用到这些信息。
　　对于map对象中的ops成员，存储的是map对象所支持的操作，如查询、删除和更新等，其定义如下：
```c
struct bpf_map_ops {
    struct bpf_map *(*map_alloc)(union bpf_attr *attr);
    void (*map_release)(struct bpf_map *map, struct file *map_file);
    void (*map_free)(struct bpf_map *map);
    int (*map_get_next_key)(struct bpf_map *map, void *key, void *next_key);
    void *(*map_lookup_elem)(struct bpf_map *map, void *key);
    int (*map_update_elem)(struct bpf_map *map, void *key, void *value, u64 flags);
    int (*map_delete_elem)(struct bpf_map *map, void *key);
    void *(*map_fd_get_ptr)(struct bpf_map *map, struct file *map_file, int fd);
    void (*map_fd_put_ptr)(void *ptr);
};
```
　　并不是所有类型的map都需要实现全部的操作，因为有些操作是特定于map类型的。而这些操作是和内部存储结构实现密切相关的，同一种操作虽然对外提供的操作接口相同，但是对于不同的存储结构其实现方式也是不一样的。struct bpf_map对象对所有enum bpf_map_type中定义的map类型都适用。
　　BPF_MAP_TYPE_HASH类型的map，顾名思义，就是内部存储结构采用hash表实现的map。下面会讲述hash表的定义，以及基于hash表实现的map所提供的操作的实现。
　　接下来看下hash对象的定义，对象中每个字段的含义都进行了注释：
```c
struct bpf_htab {
    struct bpf_map map;  /* 记录了hash table的所对应的map信息*/
    struct bucket *buckets;  /* 指向hash表中的所有桶组成的数组*/
    void *elems;  /* 指向hash entries组成的数组首地址 */
    /* 所有的hash entry都会挂载在percpu类型的链表上 */
    struct pcpu_freelist freelist; 
    /*
     * 当map type不是BPF_MAP_TYPE_PERCPU_HASH时，用来存储在每个
     * cpu上申请的一个额外的hash entry
     */
    void __percpu *extra_elems;
    atomic_t count;	/* number of elements in this hashtable */
    u32 n_buckets;	/* number of hash buckets */
    u32 elem_size;	/* size of each element in bytes */
};
```
　　struct bpf_htab对象中的elems成员指向存放hash表中所有元素的内存。hash表中元素的定义如下：
```c
struct htab_elem {
    union {
        struct hlist_node hash_node;
        struct bpf_htab *htab;
        struct pcpu_freelist_node fnode;
    };
    union {
        struct rcu_head rcu;
        enum extra_elem_state state;
    };
    u32 hash;  /* hash element对应的hash值 */
    char key[0] __aligned(8);  /* key + value组成的柔性数组 */
};
```
　　struct htab_elem对象中的hash成员即为由key值进行hash之后得到的，而key成员这个柔性数组存放的就是“key-value”信息，value值紧跟在key值之后。这个柔性数组的类型是char，这也正好应证了文档开头所说的在map内部key和value都是当作二进制数据来存储的，而这正也是map实现通用性存储的地方。
　　struct bpf_htab对象中的freelist这个成员是用来管理hash表中的所有的元素的，即所有的元素会均分到这个percpu类型的链表中，上面struct htab_elem定义中第一个联合体成员fnode正是使用freelist链表来管理元素的关键，这部分的执行流程可以参考文件kernel/bpf/hashtab.c中的函数prealloc_elems_and_freelist()。
　　在hash表的每个桶(bucket)中，使用链表来管理落在这个桶中的元素，所以bucket对象的定义如下：
```c
struct bucket {
    struct hlist_head head; /* 管理桶中所有元素的链表 */
    raw_spinlock_t lock;
};
```
　　在文档前面说过，map一个通用的key-value存储结构，所以需要支持一些操作，如插入数据、删除数据、更新数据以及查询数据等等。因此，实现一种map，除了定义其内部数据存储方式之外，还需要依据内部的数据存储方式实现其所支持的一些操作。对于struct bpf_map_ops对象中定义的操作，BPF_MAP_TYPE_HASH类型的map根据需要实现了其中的部分操作，如下：
```c
static const struct bpf_map_ops htab_ops = {
    .map_alloc = htab_map_alloc,
    .map_free = htab_map_free,
    .map_get_next_key = htab_map_get_next_key,
    .map_lookup_elem = htab_map_lookup_elem,
    .map_update_elem = htab_map_update_elem,
    .map_delete_elem = htab_map_delete_elem,
};
```
　　对于htab_ops中实现的部分操作，其具体实现可以参考文件kernel/bpf/hashtab.c，直接从代码里面看其实现，因为有上下文参考，所以更具有连贯性。

** 本文作者： TitenWang **
** 本文链接： https://titenwang.github.io/2017/03/26/ebpf-map/ **
** 版权声明： 本博客所有文章除特别声明外，均采用[ CC BY-NC-ND 3.0 ](https://creativecommons.org/licenses/by-nc-nd/3.0/cn/)许可协议。**