---
title: Redis AE异步事件库实例分析
date: 2023-11-12 12:40:21
tags: technology
---

> 注：这是2018年2月写的旧博文，转载到此。

Redis使用了一个称为“A simple event-driven programming library”的自制异步事件库(以下简称“AE”)。整个事件库的代码量少于1k行，是个优秀的C异步事件库学习材料。

# 源码结构

> 版本 Redis 4.0.8

redis的src目录下，ae开头的几个文件就是AE事件库的源码。

| 文件 | 用途 | 
|- | - |
|ae.h| AE事件库接口定义 |
|ae.c| AE事件库实现 |
|ae_epoll.c| epoll绑定 |
|ae_evport.c| evport绑定|
|ae_kqueue.c| kqueue绑定 |
|ae_select.c| select绑定 |

文件数量有点多，我们把IO多路复用的绑定都“精简”掉。在“ae.c”的开头有这么一段代码：
```
/* Include the best multiplexing layer supported by this system.
 * The following should be ordered by performances, descending. */
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```
Redis根据所处系统的不同，包含不同的IO多路复用实现代码。每种IO多路复用都实现了以下的接口：
```
struct aeApiState;
static int aeApiCreate(aeEventLoop *eventLoop);
static int aeApiResize(aeEventLoop *eventLoop, int setsize);
static void aeApiFree(aeEventLoop *eventLoop);
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask);
static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int mask);
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp);
static char *aeApiName(void);
```
任意的IO多路复用技术，只要封装出以上的接口就可以被AE事件库作为底层实现使用。我们“精简”掉4个IO多路复用绑定代码之后，就剩下“ae.h、ae.c”这两个文件了。

# AE事件模型
AE异步事件库支持以下的事件类型：

- 文件事件
- 定时器事件

Redis本身是一个KV数据库，主要就是接收客户端的查询请求并返回结果，以及对KV数据的有效性维护。所以文件事件(IO事件)和定时器事件就足以支撑服务端的全部功能。

#### 文件(IO)事件
与文件事件相关的定义和接口有：
```
#define AE_NONE 0
#define AE_READABLE 1
#define AE_WRITABLE 2

typedef void aeFileProc(struct aeEventLoop *eventLoop, int fd, void *clientData, int mask);

/* File event structure */
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE) */
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;

int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask, aeFileProc *proc, void *clientData);
void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask);
```

#### 定时器事件
与定时器事件相关的定义和接口有：
```
typedef int aeTimeProc(struct aeEventLoop *eventLoop, long long id, void *clientData);
typedef void aeEventFinalizerProc(struct aeEventLoop *eventLoop, void *clientData);

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

long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc);
int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id);
```
每种事件类型都定义了回调函数、事件结构体、事件添加/删除接口，以支持该类型事件的操作。

# AE异步事件库的典型用法
下面的例子使用AE事件库进行网络读写和定时器操作：
```
/* file: example-libae.c */
#include <stdio.h>
#include <unistd.h>
#include <time.h>
#include <ae.h>

static const int MAX_SETSIZE = 64;
static const int MAX_BUFSIZE = 128;

static void
file_cb(struct aeEventLoop *eventLoop, int fd, void *clientData, int mask)
{
    char buf[MAX_BUFSIZE] = {0};
    int rc;

    rc = read(fd, buf, MAX_BUFSIZE);
    if (rc < 0)
    {
        aeStop(eventLoop);
        return;
    }
    else
    {
        buf[rc - 1] = '\0';  /* 最后一个字符是回车 */
    }
    printf("file_cb, read %s, fd %d, mask %d, clientData %s\n", buf, fd, mask, (char *)clientData);
}

static int
timer_cb(struct aeEventLoop *eventLoop, long long id, void *clientData)
{
    printf("timer_cb, timestamp %ld, id %lld, clientData %s\n", time(NULL), id, (char *)clientData);
    return (5 * 1000);
}

static void
timer_fin_cb(struct aeEventLoop *eventLoop, void *clientData)
{
    printf("timer_fin_cb, timestamp %ld, clientData %s\n", time(NULL), (char *)clientData);
}

int main(int argc, char *argv[])
{
    aeEventLoop *ae;
    long long id;
    int rc;

    ae = aeCreateEventLoop(MAX_SETSIZE);
    if (!ae)
    {
        printf("create event loop error\n");
        goto err;
    }

    /* 添加文件IO事件 */
    rc = aeCreateFileEvent(ae, STDIN_FILENO, AE_READABLE, file_cb, (void *)"test ae file event");

    /* 添加定时器事件 */
    id = aeCreateTimeEvent(ae, 5 * 1000, timer_cb, (void *)"test ae time event", timer_fin_cb);
    if (id < 0)
    {
        printf("create time event error\n");
        aeDeleteEventLoop(ae);
        goto err;
    }

    aeMain(ae);

    aeDeleteEventLoop(ae);
    return (0);

err:
    return (-1);
}
```
可以把这个文件放在Redis的deps/hiredis/examples目录下，修改hiredis目录的Makefile
```
AE_DIR=/path/to/redis/src
example-libae: examples/example-libae.c $(STLIBNAME)
    $(CC) -o examples/$@ $(REAL_CFLAGS) $(REAL_LDFLAGS) -I. -I$(AE_DIR) $< $(AE_DIR)/ae.o $(AE_DIR)/zmalloc.o $(AE_DIR)/../deps/jemalloc/lib/libjemalloc.a -pthread $(STLIBNAME)
```
编译执行看一下效果：
```
$ make example-libae
$ examples/example-libae
123456
file_cb, read 123456, fd 0, mask 1, clientData test ae file event
timer_cb, timestamp 1519137082, id 0, clientData test ae time event
^C
```
可以看到AE异步事件库的使用还是比较简单，没有复杂的概念和接口。

# AE事件循环
一个异步事件库除了事件模型外，最重要的部分就是事件循环了。来看看AE的事件循环“aeMain”是怎么实现的：
```
/* file: ae.c */
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) { /* 运行状态判断 */
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop); /* 事件循环前回调执行 */
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```
看来“aeProcessEvents”函数才是事件循环的主体：
```
/* file: ae.c */

int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* 如果啥事件都不关注，立即返回 */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

       /* 略去计算tvp的代码 */
        …… ……

        /* 调用IO多路复用接口 */
        numevents = aeApiPoll(eventLoop, tvp);

        /* 略去IO多路复用后回调执行代码 */
        …… ……

        for (j = 0; j < numevents; j++) { /* 循环处理IO事件 */
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

            /* 执行读回调函数 */
            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask); 
            }
            /* 执行写回调函数 */
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask); 
            }
            processed++;
        }
    }
    /* 执行定时器事件 */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```
在“aeProcessEvents”函数主体中，对于文件(IO)事件的处理逻辑已经比较清晰，调用IO多路复用接口并循环处理返回的有效文件描述符。定时器事件在“processTimeEvents”函数中处理：
```
/* file: ae.c */

static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te, *prev;
    long long maxId;
    time_t now = time(NULL);

    /* 发现系统时间修改过，为防止定时器永远无法执行，将定时器设置为立即执行 */
    if (now < eventLoop->lastTime) {
        te = eventLoop->timeEventHead;
        while(te) {
            te->when_sec = 0;
            te = te->next;
        }
    }
    eventLoop->lastTime = now;

    prev = NULL;
    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    while(te) {
        long now_sec, now_ms;
        long long id;

        /* 略去删除失效定时器代码 */
        …… ……
 
        /* 略去作者都注释说没什么用的代码囧 */
        …… ……

        aeGetTime(&now_sec, &now_ms);
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;

            id = te->id;
            /* 执行回调函数 */
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;
            if (retval != AE_NOMORE) {
                /* 需要继续保留的定时器，根据回调函数的返回值重新计算延迟时间 */
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                /* 无需保留的定时器，打上删除标识 */
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        prev = te;
        te = te->next;
    }
    return processed;
}
```
AE库的定时器事件很“简单粗暴”的使用了链表。新事件都往表头添加(详见“aeCreateFileEvent”函数实现)，循环遍历链表执行定时器事件。没有使用复杂的时间堆和时间轮，简单可用，只是查找和执行定时器事件的事件复杂度都是O(n)。作者在注释中也解释说，现在基于链表的定时器事件处理机制已经足够Redis使用：
```
/* Search the first timer to fire.
 * This operation is useful to know how many time the select can be
 * put in sleep without to delay any event.
 * If there are no timers NULL is returned.
 *
 * Note that's O(N) since time events are unsorted.
 * Possible optimizations (not needed by Redis so far, but...):
 * 1) Insert the event in order, so that the nearest is just the head.
 *    Much better but still insertion or deletion of timers is O(N).
 * 2) Use a skiplist to have this operation as O(1) and insertion as O(log(N)).
 */
```

# 总结
从前面的分析可以看到，AE异步事件库本身的实现很简洁，却支撑起了业界最流行的KV内存数据库的核心功能。让我想起了业界的一句鸡汤，“要么架构优雅到不怕Bug，要么代码简洁到不会有Bug”。

# 参考
[1] [事件库之Redis自己的事件模型-ae](https://my.oschina.net/u/917596/blog/161077)，by C_Z

版权声明：自由转载-非商用-非衍生-保持署名（[创意共享4.0许可证](https://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh-hans)）