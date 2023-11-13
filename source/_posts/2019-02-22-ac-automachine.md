---
title: AC 自动机
date: 2019-02-22 16:11:42
tags:
categories: Program
---
AC自动机算法， 全称 Aho–Corasick 算法， 是由 Alfred V. Aho 和 Margaret J.Corasick 发明的字符串搜索算法。
该算法主要依靠构造一个有限状态机，在一个 trie 树中添加失配指针来实现。这些额外的失配指针允许在查找字符串失败时进行回退，转向某前缀的其他分支，免于重复匹配前缀，提高算法效率。 当一个字典串集合是已知的情况下，算法的时间复杂度为输入字符串长度和匹配数量之和。

现实中用到AC自动机的场景有如下：
* 评论反垃圾：通过维护一个屏蔽词库，建立AC自动机，输入评论，如果命中了屏蔽词，就可以把它定性为反垃圾
* 病毒检测：通过维护一个病毒码库，简历AC自动机，输入一串码，判断是否命中病毒

算法关键是
### 构建失败指针:

* 将根结点的所有孩子结点的 failPtr 指向根结点，然后将根结点的所有孩子结点依次入列。
* 若队列不为空：
  * A: 出列，出列结点记为 curr， failTo 表示 curr 的 fail 指向的结点，即 failTo = curr.fail
  * B: 判断 curr.child[i] == failTo.child[i]是否成立，
     *  成立：curr.child[i].fail = failTo.child[i]，
     *  不成立：判断 failTo == null是否成立
        * 成立： curr.child[i].fail == root
        * 不成立：执行failTo = failTo.fail，继续执行 B
     * curr.child[i]入列，再次执行再次执行步骤 A
* 若队列为空:结束

### 查找

遍历字符串的每个字符进行查找，从根结点开始，查看其孩子结点，看是否匹配，

* 如果不匹配则沿着他的失败指针查找，直到失败指针是根节点就重启自动机，再次查找。
* 如果经过的节点是模式字符串的结尾。意味着匹配到该模式串，并将该字符下标记录下来。

### 写了一个库

实现了AC自动机的 Go 语言版本。[goAcAutoMachine](https://github.com/zheng-ji/goAcAutoMachine), 支持中文搜索。
