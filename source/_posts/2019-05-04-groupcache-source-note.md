---
title: groupCache 源码阅读
date: 2019-05-04 12:11:42
tags: groupCache
categories: Program
---

自从写了 C++，这一年多时间是没有再碰到 Go 代码，五一期间想通过阅读 Go 源码来找回熟悉的味道。选择了 GroupCache 仔细研读，收获颇丰，记下笔记以作备忘。

###  groupCache 简介

[groupcache](https://github.com/golang/groupcache)是一个缓存库，需自己实现 main 函数，groupcache 既是服务器，也是客户端。作者称在某些方面替代 memcache，groupcache 只能 get，不能设置过期时间，只能通过lru淘汰最近最少访问的数据。

当在本地 groupcache 缓存中没有查找的数据时，通过一致性哈希，查找到该 key 所对应的 peer 服务器，再通过HTTP 协议，从 peer 服务器上获取数据。

过程中也做了一个优化：当多个客户端同时访问 cache 中不存在的 Key 时，会导致多个客户端从后端获取数据并同时插入 cache 中，相同 Key 情况下，groupcache 只会有一个客户端从后端获取数据，其他客户端阻塞，直到第一个客户端获取到数据之后，再返回给多个客户端。

![](/images/2019/groupcache.png)


### GroupCache 源码组成

groupCache 源码巧妙利用 Go 的Interface，将核心组件做了抽离：

* [lru](https://github.com/golang/groupcache/blob/master/lru/lru.go)
* [SingleFlight](https://github.com/golang/groupcache/blob/master/singleflight/singleflight.go)
* [consistenHash](https://github.com/golang/groupcache/blob/master/consistenthash/consistenthash.go)
* ByteView,Sink
* [HTTPPool](https://github.com/golang/groupcache/blob/master/http.go)
* [groupcache.go](https://github.com/golang/groupcache/blob/master/groupcache.go) 中对这些核心组件搭积木，成就了这份精彩的代码。

让我们先来探索各个组件，最后再做一次高屋建瓴的俯瞰。

#### 内存 LRU 

Least Recently Used 的缩写，即最近最少使用的页面置换算法。借助内建容器 `container/list` 及 哈希表实现 `Cache` 结构，不保证并发安全。list 的元素 `Entry` 是一个结构体，也是实际承载键值对的载体，当列表容器元素个数达到最大数量，触发用户自定义的 `OnEvicted` 函数。 

那 cache 是怎么做到 LRU？

* Add: 新增一个元素，先查找该元素是否在哈希表? 若不在，则将元素写入哈希表，并移动元素到 list 头部；如果列表长度超过 MaxEntries，则删除列表尾部元素。
* Get： 查找元素，如果找到则移动元素到 list 头部，始终保证最近使用的元素放在头部。
* Remove: 将元素从列表和哈希表一同删除, 并触发用户自定义 OnEvicted 函数。

#### 一致性哈希

目标是解决互联网中的server 的扩展和收缩，不至于导致大量的 Key 失效。核心数据结构是一个哈希表，键为Int，值为对应的 Key。

通过哈希函数把元素映射到整型索引值，对索引值升序排列，返回的节点是与这个索引值最近的顺时针节点。
默认选择的哈希算法是 crc32.CheckSumIEEE，为了使得环形列表更平衡，加入了虚拟节点，其中`replicas`

执行虚拟节点个数 。

我们来一窥其工作流程：

* Add: 添加节点到哈希表，对 Key 取 replicas 个 hash 值，将得到哈希值都映射到同个 Key，与此同时哈希值升序排列。
* Get：查找 Key 对应的节点，保证一致性，计算 Key 的 hash 值，然后在有序列表中二分查找，找到离他最近的顺时针的索引，在哈希表中返回此索引对应的 Key。

#### SingleFlight

`call` 存来调用的返回值和返回码。利用 sync.Wait 会阻塞的特性，控制并发性，直到所有任务完成才继续执行。结合 Group 结构来看就更明了，Group 有一个哈希表，其值是 `call `结构, 对于给定的 Key，保证同一时间只存在一个后端调用，这么设计的好处是在缓存穿透的时候只有一个请求访问后端，避免雪崩。

#### ByteView & Sink 

ByteView 是 groupCache 内部 value 的数据结构，结构体支持两种类型的元素，[]byte 或者 string 方式，并提供了 Copy 的方法。

Sink 是一个接口类型，包含了 `SetString`, `SetBytes` `SetProto` `view` 四种方法，每个实现该接口的 Struct 也额外实现了 SetView ,通过他们，value 会被保存成 byteview 的格式。

#### Peer & HTTPPool

peers.go 要与 http.go 结合起来看，不得不说抽象的很高级，或者说很绕。

HTTPPool 是实现了 http.Handler 和 PeerPicker 接口的节点池。他的组成如下：

* `peers` : 一致性哈希生成器，返回 Key 对应的服务器节点
* `httpGetters map[string]*httpGetter` 对于每个服务器，均定义了 httpclient，用于获取远程服务器的数据
其中 httpGetter 就是一个ProtoGetter,  它实现了Get 方法
* `BasePath` `指定外部服务器节点的请求的地址 默认是 "/_groupcache/". 通过 http.Handle 在NewHTTPPool 中注册`
* `Replicas` ：一致性哈希虚拟节点的数量

#### groupcache 获取数据流程

当客户端连上 groupcache 获取数据

* 如果本地有缓存，包括 mainCache (本机节点的缓存数据)，hotCache（属于其他节点的数据缓存的热数据）有所需要的数据，则直接返回。
* 如果没有，则通过一致性哈希函数判断这个 key 所对应的节点( PickPeer(key) )，通过 http 从这个节点上获取数据( getFromPeer )，如果这个节点上有需要的数据，则通过 http 回复给之前的那个 groupcache。
* 如果节点上也没有所需要的数据，则 groupcache 从后端数据源(数据库或者文件)获取数据，并将数据保存在本地 mainCache，并返回给客户端；其中 `getLocallly` 函数是利用 NewGroup 时传进去的 Getter，在调用这个 Getter 的 Get 函数从数据源获取数据。
* hotCache 存的是本节点没有的，但是访问频繁的数据， 避免多次网络请求， 热点数据采用10%随机概率，效果可能不理想。
* loadGroup 是singleFlight的实现，确保同一时间每个 Key 只有一次访问后端。



