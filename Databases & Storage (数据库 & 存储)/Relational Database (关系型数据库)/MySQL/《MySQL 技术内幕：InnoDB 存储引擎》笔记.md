# 1 MySQL 体系结构和存储引擎

## 1.1 定义数据库和实例

* 数据库和数据库实例是两个不同的概念
数据库：是文件的集合
实例：是程序，是真正用于操作数据库文件的

* 关于配置文件读取
MySQL 以读取到的最后一个配置文件中的参数为准

## 1.2 MySQL 体系结构

* 区别于其他数据库最重要的一个特点：插件式存储引擎

* 存储引擎是基于表的，而不是数据库。

## 1.3 MySQL 存储引擎

### 1.3.1 InnoDB 存储引擎

### 1.3.2 MyISAM 存储引擎

* 不支持事务、表锁，支持全文索引，主要面向一些 OLAP 数据库应用

* 它的缓冲池只缓存索引文件，而不缓冲数据文件，这和大多数数据库非常不同

# 2 InnoDB 存储引擎

## 2.1 InnoDB 存储引擎概述

特点：行锁设计、支持 MVCC、支持外键、提供一致性非锁定读

## 2.3 InnoDB 体系架构

InnoDB 存储引擎有多个内存块，组成一个大的内存池，负责：
* 维护所有线程/线程需要访问的多个内部数据结构
* 缓存磁盘上的数据，方便快速读取，同时在对磁盘文件的数据修改之前在这里缓存
* redo log 缓冲
* ...

后台线程的主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存是最近的数据。此外将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常时 InnoDB 能恢复到正常运行状态。

*（其他存储引擎跳过）*

### 2.3.1 后台线程

1. Master Thread

非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性。

2. IO Thread

主要负责大量使用了 AIO 的写 IO 请求的回调

4 个 IO Thread：write、read、insert buffer、log IO Thread

3. Purge Thread

回收已经使用并分配的 undo 页

4. Page Cleaner Thread

将之前版本中脏页的刷新操作都放入到单独的线程中完成，目的是为了减轻原 Master Thread 的工作以及对于用户查询线程的阻塞 -> 进一步提高性能

### 2.3.2 内存

1. buffer pool (缓冲池)

一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响（协调 CPU 速度与磁盘速度的鸿沟）。

缓存池中缓存的数据也类型有：索引页、数据页、undo 页、insert buffer、adaptive hash index、lock info、data dictionary 等。

![alt text](image.png)

2. LRU List、Free List 和 Flush List

通常来说，数据库中的缓冲池是用 LRU 算法管理的，即最频繁使用的页在 LRU List 的前端，最少使用的页在尾端。当缓冲池不能存放新读取到的页时，首先释放 LRU List 尾端的页。

midpoint insertion strategy: InnoDB 中，缓冲池中页的大小默认为 **16KB**。InnoDB 对传统的 LRU 算法做了优化——加入了 midpoint 位置，默认在 LRU List 长度的 5/8 处。把 midpoint 之后的列表称为 old 列表，之前的列表称为 new 列表。

参数 innodb_old_blocks_pct 控制 midpoint 位置。
参数 innodb_old_blocks_time 控制页读取到 mid 位置后需要等待多久才会被加入到 LRU List 的热端。

* LRU List 管理已经读取的页的过程（Page 27）
* 压缩页的内存分配过程（Page 29）

* 脏页（dirty page）：LRU 列表中的页被修改后的为脏页，即缓冲池中的页和磁盘上的页的数据产生了不一致。
-> 这时 DB 会通过 CHECKPOINT 机制将脏页刷新回磁盘，而 Flush List 中的页就是脏页列表。

注意：脏页既存在于 LRU List，也存在于 Flush List。LRU List 管理缓冲池中页的可用性，Flush List 管理将页刷新回磁盘，而这互不影响。

3. redo log buffer (重做日志缓冲)

InnoDB 先将 redo log 信息放入缓冲区，然后按一定频率刷新到 redo log 文件。

参数 innodb_log_buffer_size 控制每秒产生的事务量（默认 8MB）

在以下三种情况会将 redo log buffer 中的内容刷新到外部磁盘的 redo log 文件中：
* Master Thread 每一秒将 redo log buffer 刷新到 redo log 文件
* 每个事务提交时会将 redo log buffer 刷新到 redo log 文件
* 当 redo log buffer pool 剩余空间小于 1/2 时，redo log buffer 刷新到 redo log 文件

4. 额外的内存池

InnoDB 通过内存堆（heap）方式管理内存

*（20250614 Page 31 下面一段暂时不太理解）*

## 2.4 Checkpoint 技术

* Write Ahead Log 策略：当事务提交时，先写 redo log，再修改页。
-> 当发生宕机时，通过 redo log 完成数据恢复 -> ACID 的 D(Durability) 要求

Checkpoint 技术的目的是解决：
* 缩短数据库的恢复时间
* buffer pool 不够用时，将脏页刷新到磁盘
* redo log 不可用时，刷新脏页

当数据库宕机时，数据库不需要重做所有的日志，因为 Checkpoint 之前的页都已经刷新回磁盘 -> 因此 DB 只需对 Checkpoint 后的 redo log 进行恢复 -> 大大缩短恢复时间

另外当缓冲池不够用时，根据 LRU 算法溢出最少最近使用的页，此时：
如果此页为脏页 -> 需要强制执行 Checkpoint，将该脏页刷回磁盘

InnoDB 通过 LSN(Log Sequence Number) 来标记版本，8 bytes 的数字。每个页有 LSN，redo log 中也有 LSN，Checkpoint 也有 LSN。

InnoDB 内部有两种 Checkpoint：
* Sharp Checkpoint（发生在数据库关闭时）
* Fuzzy Checkpoint

*（TODO：补充 P34~P61 笔记）*

# 3 文件

## 3.1 参数文件

### 3.1.2 参数类型

* 动态（dynamic）参数
* 静态（static）参数

## 3.2 日志文件

常见日志文件类型：
* error log
* binlog
* slow query log
* log（查询日志）

### 3.2.1 error log (错误日志)

对 MySQL 的启动、运行、关闭过程进行了记录。DBA 遇到问题时应该首先查看该文件以定位问题。

### 3.2.2 slow query log (慢查询日志)

参数 long_query_time（默认 10）

默认不开启，需要手动开启，将以下参数设为 ON：
```
SHOW VARIABLES LIKE 'log_slow_queryies';
```

*（TODO：补充 P68~ 笔记）*