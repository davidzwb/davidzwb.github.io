# 自己写一个简单 Key-Value Store 之 Memcached 研究

这个小项目的目标是仿照 Memcached，用 c++ 重写一个类似的简单内存 Key-Value Store。

## Memcached

首先，我们来研究下 Memcached 的结构，Memcached 是使用 C 写的：

1. 网络模型：

   Memcached 使用 libevent 网络库，采用同步 I/O，事件驱动的形式。

   虽然采用的 libevent，但却并未使用异步 I/O，而是注册事件，可读或可写事件触发后，lidbevent 调用预先注册的回调函数，在自行进行 I/O 读写。

2. Hash Table：

   ```c
   /* how many powers of 2's worth of buckets we use */
   static unsigned int hashpower = 16;

   typedef struct _stritem {
       struct _stritem *next;
       struct _stritem *prev;
       struct _stritem *h_next;    /* hash chain next */
       rel_time_t      time;       /* least recent access */
       rel_time_t      exptime;    /* expire time */
       int             nbytes;     /* size of data */
       unsigned short  refcount;
       uint8_t         nsuffix;    /* length of flags-and-length string */
       uint8_t         it_flags;   /* ITEM_* above */
       uint8_t         slabs_clsid;/* which slab class we're in */
       uint8_t         nkey;       /* key length, w/terminating null and padding */
       uint64_t        cas_id;     /* the CAS identifier */
       void * end[];
       /* then null-terminated key */
       /* then " flags length\r\n" (no terminating null) */
       /* then data with terminating \r\n (no terminating null; it's binary!) */
   } item;

   static item** primary_hashtable = calloc(hashsize(hashpower), sizeof(void *));
   ```

   * Hash Table 默认的 bucket 大小为 2^16 个。

   * 每个 bucket 存储一个 struct item 的指针。

   * 采用 separate chaining(分离链接) 的方式来解决冲突，并且每个链表的节点和 Key-Value 的值存储在一起，这样存储在一起应该是为了提高 CPU 缓存的命中率，提高访问速度。

   * 每个 item 内有 prev 和 next 指针形成双向链表。

      \* item | \* item | \* item....

     ​      |

     ​      v

     ​    item

     ​      |

     ​      v

     ​    item

3. 内存管理：

   这里的内存管理指的是，上述的 item 结构在内存中怎么存放。

   memcached 通过三个结构来存放 item：

   page: 内存申请与分配的基本单位

   slab: item 存放的实际结构，每个 slab 都为 一个 page 的大小，每个 slab 又分为一个或多个 chunk

   slab class: 决定这个类型的 slab 内部会分为多少个 chunk

   chunk:  chunk 的大小由 slab class 决定

   ```
   $ ./memcached -vv
   slab class   1: chunk size        80 perslab   13107
   slab class   2: chunk size       104 perslab   10082
   slab class   3: chunk size       136 perslab    7710
   slab class   4: chunk size       176 perslab    5957
   slab class   5: chunk size       224 perslab    4681
   slab class   6: chunk size       280 perslab    3744
   slab class   7: chunk size       352 perslab    2978
   slab class   8: chunk size       440 perslab    2383
   slab class   9: chunk size       552 perslab    1899
   slab class  10: chunk size       696 perslab    1506
   [...etc...]
   ```
   可以看到，通过这样的设计，既可以到达预分配内存，同时又不浪费太多的空间，每个 item 都放到适合其大小的 slab class 中。