# Redis源码探究-事件驱动网络编程-Server 

> 本文使用的是 github 上 Redis 最早的源代码，Redis 1.3.6，发布于 2010 年。

Redis 使用了事件驱动网络编程，Reactor 模式，其核心是：注册事件，提供回调，非阻塞 IO。

## EventLoop

事件驱动的核心是 EventLoop 结构，它代表了一个 Event Loop，也就是说，使用者向这个 EventLoop 注册事件，并提供回调函数，EventLoop 就不停地 "Loop" 着等待事件 Event 发生，然后调用回调函数。

```c
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;
    long long timeEventNextId;
    aeFileEvent events[AE_SETSIZE]; /* Registered events */
    aeFiredEvent fired[AE_SETSIZE]; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
} aeEventLoop;
```

### 事件

抽象上来看，一个事件包含了三个部分，事件触发条件、事件处理方法以及存放事件处理结果的位置。

具体实现上来看，Redis 将事件分为了两类，FileEvent 和 TimeEvent：

1. FileEvent：

   ```c
   /* File event structure */
   typedef struct aeFileEvent {
       int mask; /* one of AE_(READABLE|WRITABLE) */
       aeFileProc *rfileProc;
       aeFileProc *wfileProc;
       void *clientData;
   } aeFileEvent;
   ```

   mask：代表事件的触发条件，分为 READABLE 或 WRITABLE。

   rfileProc/wfileProc：代表事件的处理方法，READABLE 的事件则将处理的函数存在 rfileProc，WRITABLE 的函数则将处理的函数存在 wfileProc。

   clientData：处理结果存放的内存空间。

2. TimeEvent：

   ```c
   /* Time event structure */
   typedef struct aeTimeEvent {
       long long id; /* time event identifier. */
       long when_sec; /* seconds */
       long when_ms; /* milliseconds */
       aeTimeProc *timeProc;
       aeEventFinalizerProc *finalizerProc;
       void *clientData;
       struct aeTimeEvent *next;
   } aeTimeEvent;
   ```

   timeProc：计时到后调用的处理函数。

   clientData：处理结果存放的内存空间。

   next：所有的 TimeEvent 形成一个链表。

### 事件触发

Redis 的事件触发机制是，先调用系统的 fd 监听 API，监听注册的事件 fd，当事件触发后，将触发事件统一存放到一处（fired 数组），然后集中地一个一个处理触发的事件。

Redis 支持多种 fd 监听 API，比如 select 和 epoll，而每个 API 对应的数据结构不同，所以 Redis 使用一个 void 指针来指向这些数据结构。

### aeEventLoop

aeEventLoop 结构包含了一个 Event Loop 所需要的信息。

1. maxfd：所有事件都是由一个文件描述符 fd 表示的，在注册事件时，需要提供该事件的 fd，aeEventLoop.maxfd 代表的是所有 fd 中的最大值。

2. events：存放所有注册的 FileEvent 的信息。

3. fired：存放已发生的事件。

   ```c
   /* A fired event */
   typedef struct aeFiredEvent {
       int fd;
       int mask;
   } aeFiredEvent;
   ```

   fd：已发生事件的 fd。

   mask：事件的触发条件，即 READABLE 或 WRITABLE。

4. timeEventHead：存放所有的 TimeEvent 的链表。

5. stop：EventLoop 运行状态标志位。

6. apidata：fd 监听 API 各自的具体数据。

## Server 实际启动运行

### main()

从 Server 的 main() 函数开始，只关注网络方面的代码。

在去掉一些错误处理和日志代码后，main() 函数非常简洁： 

```c
static struct redisServer server; /* server global state */

int main(int argc, char **argv) {
    initServerConfig();//初始化 Server 默认配置
    initServer();//初始化 Server
    aeMain(server.el);//执行 EventLoop
    aeDeleteEventLoop(server.el);//结束 EventLoop
    return 0;
}
```

### redisServer 结构 

redisServer 是存放 Server 所有信息的结构体，从 redisServer 中网络相关的变量如下：

```c
struct redisServer {
  int port;
  int fd;
  char neterr[ANET_ERR_LEN];
  aeEventLoop *el;
  char *bindaddr;
}
```

### initServer()

同样只提取出网络相关代码：

```c
server.el = aeCreateEventLoop();
server.fd = anetTcpServer(server.neterr, server.port, server.bindaddr); //start listening on this fd.

aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL);
aeCreateFileEvent(server.el, server.fd, AE_READABLE, acceptHandler, NULL);
```

1. aeCreateEventLoop()

   创建 EventLoop：zai

   ```c
   /* Include the best multiplexing layer supported by this system.
    * The following should be ordered by performances, descending. */
   #ifdef HAVE_EPOLL //优先选择 epoll，其次 kqueue，最次 select。
   #include "ae_epoll.c"
   #else
       #ifdef HAVE_KQUEUE
       #include "ae_kqueue.c"
       #else
       #include "ae_select.c"
       #endif
   #endif

   aeEventLoop *aeCreateEventLoop(void) {
       aeEventLoop *eventLoop;
       int i;

       eventLoop = zmalloc(sizeof(*eventLoop));
       if (!eventLoop) return NULL;
       eventLoop->timeEventHead = NULL;
       eventLoop->timeEventNextId = 0;
       eventLoop->stop = 0;
       eventLoop->maxfd = -1;
       eventLoop->beforesleep = NULL;
       if (aeApiCreate(eventLoop) == -1) {
           zfree(eventLoop);
           return NULL;
       }
       /* Events with mask == AE_NONE are not set. So let's initialize the
        * vector with it. */
       for (i = 0; i < AE_SETSIZE; i++)
           eventLoop->events[i].mask = AE_NONE;
       return eventLoop;
   }
   ```

   首先 malloc 一个 aeEventLoop 结构，然后把变量都设为初始值。

   重点在于 aeApiCreate(eventLoop)，该函数有三个实现，分别使用 epoll、kqueue 以及 select 实现，redis 使用宏来控制选择哪个版本，我主要关注性能最好的 epoll 实现：

   ```c
   #include <sys/epoll.h>

   typedef struct aeApiState {
       int epfd;
       struct epoll_event events[AE_SETSIZE];
   } aeApiState;

   static int aeApiCreate(aeEventLoop *eventLoop) {
       aeApiState *state = zmalloc(sizeof(aeApiState));

       if (!state) return -1;
       state->epfd = epoll_create(1024); /* 1024 is just an hint for the kernel */
       if (state->epfd == -1) return -1;
       eventLoop->apidata = state;
       return 0;
   }
   ```

   可以看到 eventLoop->apidata 中存放的是 epoll_create() 后生成的 epoll fd 以及 epoll_event。

2. anetTcpServer()

   根据 server.neterr、server.port 和 server.bindaddr 这三个设置，建立 socket，绑定地址，并开始监听。

   其中 port 和 bindaddr 在 initServerConfig() 中进行初始化赋值：

   ```c
   #define REDIS_SERVERPORT        6379    /* TCP port */

   server.port = REDIS_SERVERPORT;
   server.saveparams = NULL;
   ```

   值得注意的是，代码中也像我之前的文章中写到过的一样，设置了 SO_REUSEADDR 属性，使得一个地址即使在 time_wait 状态，也可以被新的 socket 绑定。

   ```c
   if (setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) == -1) {
           anetSetError(err, "setsockopt SO_REUSEADDR: %s\n", strerror(errno));
           close(s);
           return ANET_ERR;
    }
   ```

3. aeCreateFileEvent()

   ```c
   int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
           aeFileProc *proc, void *clientData);
   ```

   这个函数首先要从逻辑上理解，它的功能是创建一个 FileEvent，需要提供的参数是：

   - eventloop，创建的 FileEvent 要加入这个 eventloop；
   - fd，这个 FileEvent 要监听这个 fd 上的事件；
   - mask，这个 FileEvent 触发的条件，READABLE 或 WRITABLE；
   - proc，这个 FileEvent 触发后的处理方法；
   - clientData，这个 FileEvent 处理后数据存放的位置。

   我们再来看实际的代码实现：

   ```c
   int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
           aeFileProc *proc, void *clientData)
   {
       if (fd >= AE_SETSIZE) return AE_ERR;
       aeFileEvent *fe = &eventLoop->events[fd];

       if (aeApiAddEvent(eventLoop, fd, mask) == -1)//将 fd 加入 eventloop 监听
           return AE_ERR;
       fe->mask |= mask;//事件触发条件存入 aeFileEvent
       if (mask & AE_READABLE) fe->rfileProc = proc;//事件处理方法存入 aeFileEvent
       if (mask & AE_WRITABLE) fe->wfileProc = proc;//事件处理方法存入 aeFileEvent
       fe->clientData = clientData;//事件处理结果存入 aeFileEvent
       if (fd > eventLoop->maxfd)
           eventLoop->maxfd = fd;
       return AE_OK;
   }

   //以下为epoll 版本，还有 kqueue 和 select 版本。
   static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
       aeApiState *state = eventLoop->apidata;
       struct epoll_event ee;
       /* If the fd was already monitored for some event, we need a MOD
        * operation. Otherwise we need an ADD operation. */
       int op = eventLoop->events[fd].mask == AE_NONE ?
               EPOLL_CTL_ADD : EPOLL_CTL_MOD;

       ee.events = 0;
       mask |= eventLoop->events[fd].mask; /* Merge old events */
       if (mask & AE_READABLE) ee.events |= EPOLLIN;//READABLE转换为epoll特有的EPOLLIN
       if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;//WRITABLE转换为epoll特有的EPOLLOUT
       ee.data.u64 = 0; /* avoid valgrind warning */
       ee.data.fd = fd;//需要监听触发的 fd
       if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
       return 0;
   }
   ```

   看到这里，我们可以总结出，Redis 1.3.6 **事件触发**和**事件处理**两项上，是*割裂*的：

   * 事件触发由 eventloop->apidata 处理；
   * 事件处理由 eventloop->events[] 处理；
   * 它们之间的接触点则是事件的 fd；
   * 而已触发的事件的 fd，则由 api 函数保存到 fired[] 中；
   * 获取到已触发的事件的 fd 后，以此 fd 为 key，可以从 events[fd] 中取出事件的处理方法等信息，并进行处理。

   这样的割裂，我想是为了支持多种事件触发 API 而设计的。

   ```c
   aeCreateFileEvent(server.el, server.fd, AE_READABLE, acceptHandler, NULL);
   ```

   Redis 实际上是把刚刚创建的服务器监听 socket 加入了 eventloop，并指定触发条件为 READABLE 可读时，事件的处理方法则是 acceptHandler，在事件触发后进行事件处理时，就会调用 acceptHandler。

4. aeCreateTimeEvent()

   ```c
   long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
           aeTimeProc *proc, void *clientData,
           aeEventFinalizerProc *finalizerProc)；
   ```

   创建 TimeEvent 和 FileEvent 类似，但是其触发条件是时间，由 milliseconds 参数指定，也不需要指定 fd 和 mask 了。

   ```c
   long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
           aeTimeProc *proc, void *clientData,
           aeEventFinalizerProc *finalizerProc)
   {
       long long id = eventLoop->timeEventNextId++;
       aeTimeEvent *te;

       te = zmalloc(sizeof(*te));
       if (te == NULL) return AE_ERR;
       te->id = id;
       aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
       te->timeProc = proc;
       te->finalizerProc = finalizerProc;
       te->clientData = clientData;
       te->next = eventLoop->timeEventHead;
       eventLoop->timeEventHead = te;
       return id;
   }
   ```

   TimeEvent 的创建不需要调用 eventloop->apidata，而只是把要触发的时间保存在 aeTimeEvent 结构里了，这是可以理解的，因为 epoll 或 select 等 API 并没有提供以时间触发的条件，但是提供了时间限制，我们可以在之后看到 Redis 是如何实现的。

### aeMain(server.el)

初始化完成后，就开始正式运行 eventloop 了。

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```

当 stop 标志位为 0 时，维持运行，进入 aeProcessEvents()。

```c
#define AE_FILE_EVENTS 1
#define AE_TIME_EVENTS 2
#define AE_ALL_EVENTS (AE_FILE_EVENTS|AE_TIME_EVENTS)
#define AE_DONT_WAIT 4

int aeProcessEvents(aeEventLoop *eventLoop, int flags);
```

抽象地看，aeProcessEvents() 函数的功能是”执行指定 eventloop 中指定类型的 event”，flags 参数即用来指定类型，包括指定运行 FileEvent(AE_FILE_EVENTS)，运行 TimeEvent(AE_TIME_EVENTS) 以及两种都运行（AE_ALL_EVENTS），AE_DONT_WAIT 则是目前有什么事件执行什么事件并尽快返回。

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* Note that we want call select() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire. */
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            long now_sec, now_ms;

            /* Calculate the time missing for the nearest
             * timer to fire. */
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;
            tvp->tv_sec = shortest->when_sec - now_sec;
            if (shortest->when_ms < now_ms) {
                tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
                tvp->tv_sec --;
            } else {
                tvp->tv_usec = (shortest->when_ms - now_ms)*1000;
            }
            if (tvp->tv_sec < 0) tvp->tv_sec = 0;
            if (tvp->tv_usec < 0) tvp->tv_usec = 0;
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to se the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }
		......
```

从 TimeEvent 链表中找出触发时间最近的一个，并计算出距离此刻的时间，保存在 tvp 变量中，若没有 TimeEvent，则 tvp 置为 0。

```c
		......
		numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

	    /* note the fe->mask & mask & ... code: maybe an already processed
             * event removed an element that fired and we still didn't
             * processed, so we check if the event is still valid. */
            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }
    }
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```

aeApiPoll() 的作用是调用事件监听 API，等待事件触发返回：

```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    retval = epoll_wait(state->epfd,state->events,AE_SETSIZE,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    return numevents;
}
```

等待 epoll 的返回，返回的 epoll_event 结构都存放在 apidata->events。

然后根据这些 epoll_event，将触发的 FileEvent 从第 0 个开始逐个保存在 eventloop->fired[] 中，保存的信息包括 fd 和 触发条件。

由 aeApiPoll(eventLoop, tvp) 继续往下看。

这里的重点是，监听等待超时的时间设置为了 TimeEvent 的事件 tvp，也就是说，Redis 通过这种方式，实现了一个时间触发的事件，同时一并处理 FileEvent 事件，即：

1. 事件监听函数的超时时间设置为最近的 TimeEvent 事件的触发时间；
2. 函数返回后，先处理在这一段时间内触发的 FileEvent，这些已触发的事件的信息保存在 fired[] 中，从中取出 fd 和触发条件，以此 fd 为 key，从 events[fd] 中取出事件的处理函数，并调用进行处理。
3. FileEvent 处理完后，调用 processTimeEvents(eventLoop) 处理 TimeEvent。

```c
/* Process time events */
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te;
    long long maxId;

    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    while(te) {
        long now_sec, now_ms;
        long long id;

        if (te->id > maxId) {
            te = te->next;
            continue;
        }
        aeGetTime(&now_sec, &now_ms);
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;

            id = te->id;
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;
            /* After an event is processed our time event list may
             * no longer be the same, so we restart from head.
             * Still we make sure to don't process events registered
             * by event handlers itself in order to don't loop forever.
             * To do so we saved the max ID we want to handle.
             *
             * FUTURE OPTIMIZATIONS:
             * Note that this is NOT great algorithmically. Redis uses
             * a single time event so it's not a problem but the right
             * way to do this is to add the new elements on head, and
             * to flag deleted elements in a special way for later
             * deletion (putting references to the nodes to delete into
             * another linked list). */
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                aeDeleteTimeEvent(eventLoop, id);
            }
            te = eventLoop->timeEventHead;
        } else {
            te = te->next;
        }
    }
    return processed;
}
```

可以看到， Redis 遍历了整个 TimeEvent 链表，获取当前的时间，与这些 TimeEvent 的触发时间作对比，所有已经超过触发时间的 TimeEvent 统统处理，调用它们的 timeProc 处理函数。

### acceptHandler

到此我们已经了解到 Redis 怎么创建事件、等待事件触发并处理事件的了。

```c
aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL);
aeCreateFileEvent(server.el, server.fd, AE_READABLE, acceptHandler, NULL);
```

我们注意到，在初始化过程中 Redis 仅创建了两个事件，一个 FileEvent，一个 TimeEvent。

这个 TimeEvent 我们可以看出是每 1 毫秒执行一次 serverCron，应该是一些业务相关的东西，在此按下不表。

而这个 FileEvent 监听的是服务器端口的可读事件，是接收客户端连接并处理的地方：

```c
static void acceptHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd;
    char cip[128];
    redisClient *c;
    REDIS_NOTUSED(el);
    REDIS_NOTUSED(mask);
    REDIS_NOTUSED(privdata);

    cfd = anetAccept(server.neterr, fd, cip, &cport);
    if (cfd == AE_ERR) {
        redisLog(REDIS_VERBOSE,"Accepting client connection: %s", server.neterr);
        return;
    }
    redisLog(REDIS_VERBOSE,"Accepted %s:%d", cip, cport);
    if ((c = createClient(cfd)) == NULL) {
        redisLog(REDIS_WARNING,"Error allocating resoures for the client");
        close(cfd); /* May be already closed, just ingore errors */
        return;
    }
	......
}
```

首先 accept 新的连接，返回新连接的 socket fd。 

```c
static redisClient *createClient(int fd) {
    redisClient *c = zmalloc(sizeof(*c));

    anetNonBlock(NULL,fd);
    anetTcpNoDelay(NULL,fd);
    if (!c) return NULL;
    selectDb(c,0);
    c->fd = fd;
    c->querybuf = sdsempty();
    c->argc = 0;
    c->argv = NULL;
    c->bulklen = -1;
    c->multibulk = 0;
    c->mbargc = 0;
    c->mbargv = NULL;
    c->sentlen = 0;
    c->flags = 0;
    c->lastinteraction = time(NULL);
    c->authenticated = 0;
    c->replstate = REDIS_REPL_NONE;
    c->reply = listCreate();
    listSetFreeMethod(c->reply,decrRefCount);
    listSetDupMethod(c->reply,dupClientReplyValue);
    c->blockingkeys = NULL;
    c->blockingkeysnum = 0;
    c->io_keys = listCreate();
    listSetFreeMethod(c->io_keys,decrRefCount);
    if (aeCreateFileEvent(server.el, c->fd, AE_READABLE,
        readQueryFromClient, c) == AE_ERR) {
        freeClient(c);
        return NULL;
    }
    listAddNodeTail(server.clients,c);
    initClientMultiState(c);
    return c;
}
```

使用新建的客户端连接创建 redisClient 结构，这里大多是 Client 相关的东西，留待以后研究，重点关注 Server 网络编程相关的事情：

1. Redis 将新连接设置为了非阻塞 IO。
2. Redis 将新连接的 socket 属性设置为了 TcpNoDelay，如我之前的博客写过，不这样设置的话，因为 Nagle 算法的缘故，会产生不必要的延迟。
3. 调用 aeCreateFileEvent() 将新连接加入 EventLoop 进行监听，触发条件为 READABLE，处理方法为 readQueryFromClient()，处理结果存放位置为这个 redisClient 结构中。

```c
#define REDIS_IOBUF_LEN         1024

static void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    redisClient *c = (redisClient*) privdata;
    char buf[REDIS_IOBUF_LEN];
    int nread;
    REDIS_NOTUSED(el);
    REDIS_NOTUSED(mask);

    nread = read(fd, buf, REDIS_IOBUF_LEN);
    if (nread == -1) {
        if (errno == EAGAIN) {
            nread = 0;
        } else {
            redisLog(REDIS_VERBOSE, "Reading from client: %s",strerror(errno));
            freeClient(c);
            return;
        }
    } else if (nread == 0) {
        redisLog(REDIS_VERBOSE, "Client closed connection");
        freeClient(c);
        return;
    }
    if (nread) {
        c->querybuf = sdscatlen(c->querybuf, buf, nread);
        c->lastinteraction = time(NULL);
    } else {
        return;
    }
    if (!(c->flags & REDIS_BLOCKED))
        processInputBuffer(c);
}
```

当客户端的可读事件触发后，从客户端读取最多 1024 个字节，保存到 redisClient->querybuf 中，最后调用 processInputBuffer() 处理客户端的查询请求。

## 尾声

至此，Redis Server 的初始化流程就研究完了，果然写得很好啊，利用 C 完成了事件驱动的编程，并做了一层抽象，以适应不同的监听 API，学习了！

参考文章：https://redis.io/topics/internals-rediseventlib