---
title: Libco
date: 2018-07-14 16:46:46
categories: Program
---

> Libco 是 微信出品的 C/C++ 的协程库, 通过仅有的几个函数接口 co_create/co_resume/co_yield 再配合 co_poll，可以支持同步或者异步的写法

让我们用官方的 [example_cond.cpp](https://github.com/Tencent/libco/blob/master/example_cond.cpp) 来开启学习之旅

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <queue>
#include "co_routine.h"
using namespace std;
struct stTask_t
{
    int id;
};
struct stEnv_t
{
    stCoCond_t* cond;
    queue<stTask_t*> task_queue;
};
void* Producer(void* args)
{
    co_enable_hook_sys();
    stEnv_t* env = (stEnv_t*)args;
    int id = 0;
    while (true)
    {
        stTask_t* task = (stTask_t*)calloc(1, sizeof(stTask_t));
        task->id = id++;
        env->task_queue.push(task);
        printf("%s:%d produce task %d\n", __func__, __LINE__, task->id);
        co_cond_signal(env->cond);
        poll(NULL, 0, 1000);
    }
    return NULL;
}
void* Consumer(void* args)
{
    co_enable_hook_sys();
    stEnv_t* env = (stEnv_t*)args;
    while (true)
    {
        if (env->task_queue.empty())
        {
            co_cond_timedwait(env->cond, -1);
            continue;
        }
        stTask_t* task = env->task_queue.front();
        env->task_queue.pop();
        printf("%s:%d consume task %d\n", __func__, __LINE__, task->id);
        free(task);
    }
    return NULL;
}
int main()
{
    stEnv_t* env = new stEnv_t;
    env->cond = co_cond_alloc();

    stCoRoutine_t* consumer_routine;
    co_create(&consumer_routine, NULL, Consumer, env);
    co_resume(consumer_routine);

    stCoRoutine_t* producer_routine;
    co_create(&producer_routine, NULL, Producer, env);
    co_resume(producer_routine);

    co_eventloop(co_get_epoll_ct(), NULL, NULL);
    return 0;
}
```

通过以下命令编译就可以执行
```
git clone https://github.com/Tencent/libco
cd libco
make
// 需要-ldl -lpthread
g++ example_cond.cpp -o main  -L./lib -lcolib -lpthread -ldl
./main 
```

重点函数如下

* co_create 是协程创建函数, 原型如下
co 是协程控制块，attr 创建协程属性，一般用来设置共享栈模式和栈空间大小，默认为 NULL，routine 是协程的入口函数，arg 是函数的参数

```
int co_create(stCoRoutine_t** co, const stCoRoutineAttr_t* attr,
void* (*routine)(void*), void* arg);
```


* co_resume 

切换到指定的协程 co，操作系统对协程是无感知的，切换调度都是由协程自己来完成

* co_eventloop

co_eventloop 是主协程的调度函数，函数的主要作用就是通过 epoll 负责各个协程的时间监控，如果有网络事件到了或者等待时间超时了，就切换到相应的协程处理。ctx 是库函数 co_get_epoll_ct()，pfn 是一个钩子函数，用户自定义，在函数 co_eventloop 的最后会执行，arg 是 pfn的参数

```
void co_eventloop(stCoEpoll_t *ctx, pfn_co_eventloop_t pfn, void *arg)
```


* co_enable_hook_sys() 

co_enable_hook_sys 函数是用来打开 libco 的钩子标示，进行系统 io 函数的时候才会调用到 libco 的函数而不是原系统函数

*  poll 相当于我们平时用到的 sleep 函数。
如果用 sleep 作用到的是整个线程，通过 poll 来实现协程的挂起，协程环境下实际调用的函数是 co_poll，主要是完成回调函数的设置，超时事件的挂入然后把自己切出去。

```
int poll(struct pollfd fds[], nfds_t nfds, int timeout)
```

* 涉及到同步的接口有

```
co_cond_alloc
co_cond_signal
co_cond_broadcast
co_cond_timedwait
```
