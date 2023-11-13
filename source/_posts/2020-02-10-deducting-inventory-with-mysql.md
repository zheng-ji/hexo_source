---
title: MySQL 扣减库存的笔记
date: 2020-02-10 12:11:42
tags:  mysql
categories: Program
---

### 一个思考

电商网站有一个场景是：用户购买一个商品，库存会做扣减。如果用 MySQL 做商品存储，会遇到什么坑，该怎么避免。

简化的商品表如下。

```sql
CREATE TABLE `goods` (
  `id` int(10) NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `name` varchar(256) NOT NULL DEFAULT '' COMMENT '商品名称',
  `available` int(11)  NOT NULL DEFAULT '0' COMMENT '库存剩余量',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```


在阅读网上的解决方法之前，脑海会迸发一个解决方法是：

```sql
UPDATE goods SET available = available - 1 WHERE id = xxx 
```

如果是这么简单的话，那就没有必要讨论了。实际上在并发请求的业务场景下，有可能出现 available 变量变成负数。

虽然我们可以将 available 变量设计成 int unsigned 字段来避免，但是在不同的 MySQL 版本下，会出现 available 溢出成为一个非常大的数。


### 解决方法
* 悲观锁的方案：SELECT FOR UPDATE

当我们查询 goods 信息后就把当前的数据锁定，直到我们修改完毕后再解锁。不过 FOR UPDATE 仅适用于 InnoDB，且必须在事务区块中才能生效。


```
set autocommit = 0; // 设置MySQL为非autocommit模式：
begin trans;// 开始事务
select avaliable from goods where id = xxx for update; //1.查询出商品信息
if (avaliable >= 0) {
   affectNum = udpate goods set available = available - 1 where id = xx ;
   commit ; // 4.提交事务
} else {
   rollback ;
}
```

弊端：
悲观锁是排他锁，会阻塞其他写请求，如果上述代码有异常，导致 Commit/Rollback 没有执行，就会造成所有请求都在等待锁。

* 乐观锁的方案

乐观锁假设数据一般情况下不会冲突，在数据提交更新的时候，才会对检测数据的冲突与，如果发现冲突了，则让返回用户错误的信息，让用户决定。
通常做法是记录加一个 `Version` 字段。读取记录时，连同 `Version` 一同读出。数据每次更新时候，Version + 1， 更新时判断记录的当前版本与之前取出来的 Version 值比对，
如果当前版本号与第一次取出来的 Version 值相等，则予以更新，否则认为是过期数据。

```sql
select avaiable ,version from goods where id=xxx limit 1; // 查询version
if (available > 0) {
	update goods set avaiable=avaiable-1 where id =xx and version= <刚查出来的 Version>;
}
```

弊端： 这样虽然保证安全，但是需要执行2次 SQL

* Update 时增加 available 查询条件

```sql
udpate goods set available = available - 1 where id = xx and available - 1 >= 0 ;
```

Innodb 的 Update 语句，对于 id 是主键索引的情况下会执行行锁。该语句在 MySQL中是先读后更新，串行且原子的。单条语句，实际上也是一个事务。这个方法效率更快，且安全。

