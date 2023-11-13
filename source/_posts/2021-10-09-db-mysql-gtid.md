---
title: MySQL GTID 复制
date: 2021-10-08 11:30:42
tags:  DataBase
categories: DataBase
---

GTID 是基于 MySQL 服务器生成的已经被成功执行的全局事务ID，由服务器ID以及事务ID组合而成。这个全局ID在所有存在主从关系的数据库服务器上是唯一的。这样特性使 MySQL 的主从复制变得更加简单，以及数据库一致性更可靠。
 
### GTID 是什么

MySQL-5.6.5开始支持GTID。 global transaction identifiers。全局唯一ID。
一个GTID在一个服务器上只执行一次，避免重复执行导致数据混乱或者主从不一致。
GTID用来代替传统复制方法，不再使用 `MASTER_LOG_FILE` 与 `MASTER_LOG_POS` 开启复制。而是使用 `MASTER_AUTO_POSTION=1` 的方式开始复制。

### 为什么用 GTID 复制
1、更简单搭建主从， 不用以前那样在需要找log_file和log_pos。
2、比传统的复制更加安全，保证数据的一致性，零丢失。

### GTID的工作原理

1 当一个事务在主库端执行并提交时，产生GTID，记录到 binlog 。
2 binlog 传输到 slave, 存储到 slave 的 relaylog 后，设置gtid_next变量，告诉 slave，下一个要执行的 GTID 值。
3 SQL 线程从 relay log中获取GTID，然后对比 slave的 binlog 是否有该GTID。
4 如果有记录，说明该 GTID 的事务已经执行，slave 会忽略。
5 如果没有记录，slave 就会执行该 GTID 事务，并记录该 GTID 到自身的 binlog， 在执行事务前会先检查其他 session 持有该GTID，确保不被重复执行。
6 在解析过程中会判断是否有主键，如果没有就用二级索引，如果没有就用全部扫描。

### 配置GTID

对于 GTID 的配置(使用mysql-5.6.5以上版本)，如下:

* 主：

```
[mysqld]
server_id=1                  #服务器id
gtid_mode=on                 #开启gtid模式
enforce_gtid_consistency=on  #强制gtid一致性，开启后对于特定create table不被支持

#binlog
log_bin=master-binlog
log-slave-updates=1    
binlog_format=row            #其他格式可能造成数据不一致

#relay log
skip_slave_start=1            
```

* 从：

```
[mysqld]
gtid_mode=on
enforce_gtid_consistency=on
server_id=2

#binlog
log-bin=slave-binlog
log-slave-updates=1
binlog_format=row            #其他格式可能造成数据不一致

#relay log
skip_slave_start=1
```

### 开始同步

* 新数据库服务器, 在slave端执行以下操作

```
[master]> CHANGE MASTER TO  
    ->  MASTER_HOST='$IP',    
    ->  MASTER_USER='repl',    
    ->  MASTER_PASSWORD='xxx',    
    ->  MASTER_PORT=3306,    
    ->  MASTER_AUTO_POSITION = 1;

[master]> start slave;
Query OK, 0 rows affected (0.01 sec)
```

* 传统复制的转向 GTID复制

1. 按上文修改配置参数文件；

2. 所有服务器设置global.read_only参数，等待主从服务器同步完毕；

```
mysql> SET @@global.read_only = ON; 
```

3. 依次重启主从服务器；

4. 使用change master 更新主从配置；

```
mysql> CHANGE MASTER TO
  > MASTER_HOST = host,
  > MASTER_PORT = port,
  > MASTER_USER = user,
  > MASTER_PASSWORD = password,
  > MASTER_AUTO_POSITION = 1;
```

5.  START SLAVE;
