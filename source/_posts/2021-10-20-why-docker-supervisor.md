---
title: " Docker里使用 supervisor 管理多个程序"
date: 2021-10-20 20:20
comments: true
categories: Docker
description: Docker
---

### 问题：
* 之前在使用 Docker 有一个困惑，如果我将程序的启动命令作为启动的 `EntryPoint`, 如果服务有问题，Docker 就启不来了，更谈不上进去定位是哪里出错了。

* Docker 容器在启动的时候开启单个进程。但我们经常需要在一个机器上开启多个服务, 通常我们会用一个 shell 脚本，包罗各种启动命令来运行多个程序。

但是这样的缺点:
1. 无法看到各个子进程的运行状态
2. 如果 shell 脚本有问题，那整个 docker 也启动不了，就不给你机会登陆 docker 里面去查看具体内容。

### 精妙的方案:

如果将 Docker 的 CMD 启动命令换成启动一个 Supervisord 本身，再加上一个不休眠的程序来做为启动入口，问题就迎刃而解. 因为 Supervisord 会常驻运行, 再由它去托管各种子程序的自动拉起, 之后再加上一句 `sleep infinity`. 保证这是一个永不退出的 CMD, docker 启动后，你就可以进去里面 debug 你需要的各种程序，包括重启之类的。


```
#!/usr/bin/sh
nohup /usr/bin/supervisord >/dev/null 2>&1 &
sleep infinity
```


