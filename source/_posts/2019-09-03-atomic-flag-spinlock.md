---
title: C++11 atomic_flag 自旋锁 
date: 2019-09-03 12:11:42
tags:  C++11
categories: Program
---

C++11 可以通过 atomic_flag 来实现自旋锁。

`test_and_set()`函数检查 atomic_flag 标志, 如果之前没被设置过则设置 atomic_flag 的标志, 并返回 false;
如果之前已被设置则返回 true。

```c
class Spinlock
{
public:
    Spinlock(): flag(ATOMIC_FLAG_INIT) {}
    void lock()
    {
        while( flag.test_and_set() );
    }

    void unlock()
    {
        flag.clear();
    }

public:
    std::atomic_flag flag;
}
```

