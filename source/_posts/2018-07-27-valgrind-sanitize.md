---
layout: post
title: C++ 调试 - Valgrind, Sanitizer, GDB
date: 2018-07-14 16:46:46
categories: Program
---

本文介绍各种 C++ 的调试方法
* valgrind
* sanitizer
* gdb

`valgrind` 可用来检测 C/C++ 代码是否内存泄漏。Valgrind 使用起来不需修改目标程序源码。

用一个明显有内存泄漏的代码来演示

```
#include <stdio.h>
#include <malloc.h>

int main(void) {
    int *p = NULL;
    p = malloc(10 * sizeof(int));
    free(p);
    *p = 3;
    return 0;
}
```

编译成二进制, 然后用 valgrind 检测

```
g++ -g -o bug main.c
valgrind --tool=memcheck --leak-check=full --log-file=memchk.log ./bug
```

valgrind 会做一次 `全身检查`, 如果有内存泄漏， 我们要关注的是 LeakSummary:

```
LeakSummary
definitely lost: 确定有内存泄漏，表示在程序退出时，该内存无法回收.
indirectly lost: 间接内存泄漏，比如结构体中定义的指针指向的内存无法回收；
possibly lost： 可能出现内存泄漏，比如程序退出时，没有指针指向一块内存的首地址了
still reachable: 程序没主动释放内存，在退出时候该内存仍能访问到，比如全局 new 的对象没 delete，操作系统会回收
```

----

再推荐另一个工具 [sanitizer](https://github.com/google/sanitizers), 同样是上面的示例代码，

用 `-fsanitize=address` 编译, 然后执行二进制，会打印出错的地址空间。

```
gcc -fsanitize=address -g -o bug main.c
./bug
```

显示出错的地址空间

```
==8035== ERROR: AddressSanitizer: heap-use-after-free on address 0x60080000bfd0 at pc 0x4007e1 bp 0x7ffeb17cf780 sp 0x7ffeb17cf778
WRITE of size 4 at 0x60080000bfd0 thread T0
    #0 0x4007e0 (/data1/mm64/levizheng/QQMail/study/test_debug/a+0x4007e0)
    #1 0x7f6808337c04 (/usr/lib64/libc-2.17.so+0x21c04)
    #2 0x4006b8 (/data1/mm64/levizheng/QQMail/study/test_debug/a+0x4006b8)
```

然后用命令指定二进制来查看地址空间, `address2line -e bug`, 输入出错的地址空间，比如 `0x4007e0`, 
就可以看到他对应的代码了。 

如果用 addr2line
addr2line -C -f -e <executable_bin> 0xffc3b7

----

说回 GDB，大多数时候用的还是最多的。

用 GDB 调试一个程序

```
gdb <execute_binary>
```

如需传入启动参数，使用 `set args`，例如

```
set args -i config.file
```

开始运行则输入 `run`

当发生异常时，输入 `bt`, 就可以看到堆栈信息了。 
`f #num` 可以指定 具体frame 查看信息，`p arg` 用来打印变量值。
下面例子中, 就是查看变量的。

```
p __str
p *this
```

再次展现完整的例子:

```
Program received signal SIGSEGV, Segmentation fault.
std::basic_string<char, std::char_traits<char>, std::allocator<char> >::basic_string (this=0x7fffffffdd40, __str=...) at /data1/gcc-4.8.2_build/x86_64-unknown-linux-gnu/libstdc++-v3/include/bits/basic_string.tcc:173
173    /data1/gcc-4.8.2_build/x86_64-unknown-linux-gnu/libstdc++-v3/include/bits/basic_string.tcc: No such file or directory.
(gdb) bt
#0  std::basic_string<char, std::char_traits<char>, std::allocator<char> >::basic_string (this=0x7fffffffdd40, __str=...) at /data1/gcc-4.8.2_build/x86_64-unknown-linux-gnu/libstdc++-v3/include/bits/basic_string.tcc:173
#1  0x00000000006476d6 in GetBinLogSynShmPath (this=<optimized out>) at mmux/mmuxstatbf/mmuxstatbfconfig.h:95
#2  BinLogSyncMgr::InitShmData (this=0x1fbee80) at mmux/mmuxstatbf/binlogsyncmgr.cpp:112
#3  0x0000000000641b80 in GetMetaPtr (this=0x1fbee80) at mmux/mmuxstatbf/binlogsyncmgr.h:68
#4  MMUxStatBFServer::BeforeMasterRun (this=0x7fffffffe1d0, pvProcArgs=<optimized out>) at mmux/mmuxstatbf/mmuxstatbfserver.h:75
(gdb) p __str
$1 = (const std::basic_string<char, std::char_traits<char>, std::allocator<char> > &) @0x20: <error reading variable>
(gdb) f 1
#1  0x00000000006476d6 in GetBinLogSynShmPath (this=<optimized out>) at mmux/mmuxstatbf/mmuxstatbfconfig.h:95
95    mmux/mmuxstatbf/mmuxstatbfconfig.h: No such file or directory.
(gdb) p  *this
(gdb) f 2
#2  BinLogSyncMgr::InitShmData (this=0x1fbee80) at mmux/mmuxstatbf/binlogsyncmgr.cpp:112
112    mmux/mmuxstatbf/binlogsyncmgr.cpp: No such file or directory.
(gdb) info locals
is_new = 0
__func__ = "InitShmData"
(gdb) f 1
```

