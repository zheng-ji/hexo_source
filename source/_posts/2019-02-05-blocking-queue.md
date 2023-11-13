---
title: 实现一个阻塞队列
date: 2019-02-05 16:11:42
tags:
categories: Program
---

### STL queue 的实现
STL queue 的 API 
```
template <typename T， typename Container = std::deque<T>>
class queue
{
    T & front();
    void push(const T &elem);
    void pop();
    bool empty() const; 
};
```

API 看起来平淡自然，他的背后有设计的考量:
* 用于存储元素的底层容器，容器须满足顺序容器，如 std::vector， std::list， std::dequeue
* 在调用 front() 之前，需保证队列不为空， 即 `!empty()` ， 否则会出现未定义错误

实际应用中，我们希望队列作为一个基础组件，可以像 Redis Queue 那样
* 元素出列的操作将会阻塞，直到队列不为空， 在多线程环境下，我们通常不需要 front() 操作，而只是调用 pop() 将元素返回，操作是阻塞的
* 允许并发访问队列

### 阻塞队列实现

* 使用条件变量实现阻塞，要是队列是空的，则 pop() 操作将会阻塞队列
* 提供了一个 `try_pop()` 操作，要是队列是空的， `try_pop()` 将会立即返回而不会阻塞

```c
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
template <typename T, typename Container = std::queue<T>>
class ThreadQueue
{
    public:
        using container_type           = Container;
        using value_type               = typename Container::value_type;
        using reference                = typename Container::reference;
        using const_reference          = typename Container::const_reference;
        using size_type                = typename Container::size_type;
        using mutex_type               = std::mutex;
        using condition_variable_type  = std::condition_variable;
    private:
        Container                queue_;
        mutable mutex_type       mutex_;
        condition_variable_type  cond_;

    public:
        ThreadQueue() = default;
        ThreadQueue(const ThreadQueue &) = delete;
        ThreadQueue &operator= (const ThreadQueue &) = delete;
        void pop(reference elem)
        {
            std::unique_lock<mutex_type> lock( mutex_ );
            cond_.wait( lock, [this]() {  return !queue_.empty();  } );
            elem = std::move( queue_.front() );
            queue_.pop();
        }
        bool try_pop( reference elem )
        {
            std::unique_lock<mutex_type> lock( mutex_ );
            if ( queue_.empty() ) {
                return false;
            }
            elem = std::move( queue_.front() );
            queue_.pop();
            return true;
        }

        bool empty() const
        {
            std::lock_guard<mutex_type> lock( mutex_ );
            return queue_.empty();
        }

        size_type size() const
        {
            std::lock_guard<mutex_type> lock( mutex_ ); 
            return queue_.size();
        }

        void push( const value_type &elem )
        {
            std::lock_guard<mutex_type> lock( mutex_ );
            queue_.push( elem );
            cond_.notify_one();
        }

        void push( value_type &&elem )
        {
            std::lock_guard<mutex_type> lock( mutex_ );
            queue_.push( std::move( elem ) );
            cond_.notify_one();
        }  
};
```
