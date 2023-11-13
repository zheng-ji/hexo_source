---
title: Go Channel 笔记
date: 2021-10-13 15:30:42
tags:  Go
categories: Go
---

[toc]

Channel 是我认为 Go 最灵活的部分，而我应用的方法不多，此文是我阅读《Go并发编程实战》总结下来，当作备忘。

### Channel 是什么

可在多个 goroutine 从/往 一个Channel 中 receive/send 数据， 不必考虑额外的同步措施。Channel可以作为一个先入先出的队列，接收的数据和发送的数据的顺序是一致的。

- chanel 类型
	
buffered chann 满了，就会阻塞， 使用 make 分配结构空间及其附属空间，并完成其间的指针初始化， make 返回这个结构空间，不另外分配一个指针

```go
//带缓冲的Channel make 
ch := make(chan Task， 3)
chan T          // 可以接收和发送类型为 T 的数据
chan<- float64  // 只可以用来发送 float64 类型的数据
<-chan int      // 只可以用来接收 int 类型的数据
```
	
- 关闭 Channel 

以下代码检查是否关闭， 它可以用来检查Channel是否已经被关闭了。从Channel接收一个值，如果Channel关闭了或没有数据，那么ok将被置为false

```
close(chan)
x， ok = <- c 
```

- 几种情况下的读写 

在一个已经 close 的 unbuffered Channel上执行读操作，会返回Channel对应类型的零值，比如 bool 型 Channel 返回 false，int 型 Channel 返回0。

* 向 close的Channel写则会触发panic。读不会导致阻塞。
* 往 nil Channel 中发送数据会一直被阻塞着。
* 对一个没有初始化的Channel进行读写操作都将发生阻塞，例子如下：

| 操作	 |   空值(nil) |	已关闭	|
|  ----  | ----        | ----       |
|关闭	 |  panic	   | panic	    |
|写      |	阻塞	   | panic	    |
|读      |  阻塞	   |不阻塞      |	

Example

```
package main

func main() {
	var c chan int
	<-c
}

$ go run testnilChannel.go
fatal error: all goroutines are asleep – deadlock!

func main() {
	var c chan int
	c <- 1
}

$ go run testnilChannel.go
fatal error: all goroutines are asleep – deadlock!
```

### 代码技巧
- 与 select 的配合使用

select 语句和 switch 语句一样，它不是循环，它只会选择一个 case 来处理，如果想一直处理 Channel，你可以在外面加一个无限的for循环

- range

`range c` 产生的迭代值为Channel中发送的值，它会一直迭代直到 Channel 被关闭。上面的例子中如果把close(c)注释掉，程序会一直阻塞在 for 那一行。

```
for i := range c {
	fmt.Println(i)
}
```

###  业务使用场景

- 超时控制，心跳 HeartBeat

```
// 利用 time.After 实现
func worker(start chan bool) {
    timeout := time.After(30 * time.Second)
    for {
        select {
            // … do some stuff
            case <- timeout:
                return
        }
    }
}

// 与 timeout实现类似，下面是一个简单的心跳select实现：

func worker(start chan bool) {
    heartbeat := time.Tick(30 * time.Second)
    for {
        select {
            // … do some stuff
            case <- heartbeat:
                //… do heartbeat stuff
        }
    }
}
```

- 取最快的结果

```
main() {
    ret := make(chan string， 3)
    for i := 0; i < cap(ret); i++ {
        go call(ret)
    }
    fmt.Println(<-ret)
}

func call(ret chan<- string) {
    // do something
    // ...
    ret <- "result"
}
```

- 限制并发

```
// 最大并发数为 2
limits := make(chan struct{}， 2)
for i := 0; i < 10; i++ {
    go func() {
        // 缓冲区满了就会阻塞在这
        limits <- struct{}{}
        do()
        <-limits
    }()
}
```

- 广播， 多个 goroutine 同步响应

```
func main() {
    c := make(chan struct{})
    for i := 0; i < 5; i++ {
        go do(c)
    }
    close(c)
}

func do(c <-chan struct{}) {
    // 会阻塞直到收到 close
    <-c
    fmt.Println("hello")
}
```

- 等待一个事件

main goroutine 通过"<-c"来等待 sub goroutine中的完成事件，sub goroutine 通过close Channel触发这一事件。当然也可以通过向 Channel 写入一个 bool 值的方式来作为事件通知。main goroutine 在 Channel c上没有任何数据可读的情况下会阻塞等待。

```
import "fmt"

func main() {
    fmt.Println("Begin doing something!")
    c := make(chan bool)
    go func() {
        fmt.Println("Doing something…")
            close(c)
    }()
    <-c
    fmt.Println("Done!")
}
```

### 忘记关闭的陷阱

事实上除了超时场景，其他使用协程(goroutine)的场景，也很容易因为实现不当，导致协程无法退出，随着时间的积累，造成内存耗尽，程序崩溃。

造成泄露的例子

```
func do(taskCh chan int) {
	for {
		select {
		case t := <-taskCh:
			time.Sleep(time.Millisecond)
			fmt.Printf("task %d is done\n"， t)
		}
	}
}

func sendTasks() {
	taskCh := make(chan int， 10)
	go do(taskCh)
	for i := 0; i < 1000; i++ {
		taskCh <- i
	}
}

func TestDo(t *testing.T) {
    t.Log(runtime.NumGoroutine())
    sendTasks()
	time.Sleep(time.Second)
	t.Log(runtime.NumGoroutine())
}
```

正确的样子
```
func doCheckClose(taskCh chan int) {
	for {
		select {
		case t， beforeClosed := <-taskCh:
			if !beforeClosed {
				fmt.Println("taskCh has been closed")
				return
			}
			time.Sleep(time.Millisecond)
			fmt.Printf("task %d is done\n"， t)
		}
	}
}

func sendTasksCheckClose() {
	taskCh := make(chan int， 10)
	go doCheckClose(taskCh)
	for i := 0; i < 1000; i++ {
		taskCh <- i
	}
	close(taskCh)
}

func TestDoCheckClose(t *testing.T) {
	t.Log(runtime.NumGoroutine())
	sendTasksCheckClose()
	time.Sleep(time.Second)
	runtime.GC()
	t.Log(runtime.NumGoroutine())
}
```

---

link: http://colobu.com/2016/04/14/Golang-Channels/
link：https://studygolang.com/articles/11320
