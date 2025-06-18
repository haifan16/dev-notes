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

*（TODO：补充 Page 34~61 笔记）*

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

### 3.2.3 查询日志

general_log

### 3.2.4 二进制日志

binary log 记录了对 MySQL 数据库执行更改的所有操作，但是不包括 SELECT 和 SHOW 这类操作，因为这类操作对数据本身并没有修改。

如果用户想记录 SELECT 和 SHOW 操作，那只能使用查询日志。

二进制日志主要有以下几种作用：
* recovery
* replication
* audit(审计)：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入的攻击

参数 **max_binlog_size** 指定单个二进制日志文件的最大值，如果超过该值，则产生新的二进制日志文件，后缀名 +1，并记录到 .index 文件。

当使用事务的表存储引擎（如 InnoDB）时，所有 uncommited 的二进制日志会被记录到一个缓存中，等该事务 commited 时直接将缓冲中的二进制日志写入二进制日志文件，而该缓冲的大小由 **binlog_cache_size** 决定（默认 32K）。

默认情况下，二进制日志并不是在每次写的时候同步到磁盘（用户可以理解为缓冲写）。当数据库所在操作系统宕机时，可能会有最后一部分数据没有写入二进制日志文件中，会给恢复和复制带来问题。参数 **sync_binlog**=[N] 表示每写缓冲多少次就同步到磁盘，默认值为 0，如果将 N 设为 1，即 sync_binlog=1 表示采用同步写磁盘的方式来写二进制日志，这时写操作不适用操作系统的缓冲来写二进制日志。

参数 **innodb_support_xa** 设为 1 可以解决...（Page 77 第二段），虽然这个参数和 XA 事务有关，但它同时也能确保二进制日志和 InnoDB 存储引擎数据文件的同步。

参数 **binlog-do-db** 和 **binlog-ignore-db** 表示需要写入或忽略写入哪些库的日志，默认为空，表示需要同步所有库的日志到二进制日志。

参数 **log-slave-update**

参数 **binlog_format** 很重要，它影响了记录二进制日志的格式：（1）STATEMENT；（2）ROW；（3）MIXED。设置为 ROW 对磁盘空间要求比 STATEMENT 大很多。这项设置可以更好地保证主从数据库之间的数据一致性。

## 3.3 套接字文件（socket）

```
show variables like 'socket';
```

## 3.4 pid 文件

```
show variables like 'pid_file';
```

## 3.5 表结构定义文件

以 frm 为后缀名的文件，记录了该表的表结构定义。

frm 还用来存放视图的定义，如：用户创建了一个 v_a 视图（TODO：了解是什么），那么对应地会产生一个 v_a.frm 文件，用来记录视图的定义，是文本，可以直接 cat 查看。

## 3.6 InnoDB 存储引擎文件

### 3.6.1 表空间文件（tablespace file）

默认配置下会有一个初始大小为 10MB，名为 ibdata1 的文件。它就是默认的 tablespace file。

* 参数 innodb_data_file_path
* 参数 innodb_file_per_table：如果设置了这个参数，用户可以将每个基于 InnoDB 的表产生一个独立表空间。其命名规则为：表名.ibd

### 3.6.2 重做日志文件（redo log file）

* innodb_log_file_size
* innodb_log_files_in_group
* innodb_mirrored_log_groups
* innodb_log_group_home_dir

重做日志文件的大小对 InnoDB 的性能有非常大的影响：
一方面，不能设置得太大 -> 恢复时可能需要很长时间；
另一方面，不能设置得太小
    -> (1) 可能导致一个事务的日志需要多次切换重做日志文件
    -> (2) 会导致频繁地发生 async checkpoint -> 导致性能的抖动
        具体描述：因为重做日志有一个 capacity 变量，该值代表了最后的检查点不能超过这个阈值，如果超过则必须将 innodb buffer pool 中 flush list 中的部分脏数据写回磁盘，这时会导致用户线程的阻塞。

Q：既然同样是记录事务日志，redo log 和之前介绍的 binlog 有什么区别？
A：
1. **binlog 会记录所有与 MySQL 数据库有关的日志记录**，包括 InnoDB、MyISAM 等其他存储引擎的日志；而 InnoDB 的 redo log 只记录有关该存储引擎本身的事务日志。
2. **记录的内容不同**，无论将 binlog 文件的记录格式设为 STATEMENT、ROW、MIXED，其记录的都是关于一个事务的具体操作内容，即该日志是逻辑日志；而 InnoDB 的 redo log 记录的是关于每个页的更改的物理情况。
3. **写入的时间不同**，binlog 文件仅在事务提交前进行提交，即只写磁盘一次，不论这时该事务多大；而在事务进行的过程中，却不断有 redo entry(重做日志条目) 被写入到 redo log 文件中。

Q：重做日志条目的结构
A：
* redo_log_type 占用 1 byte
* space 表示表空间的 ID，但采用压缩的方式，因此占用的空间可能小于 4 bytes
* page_no 表示页的偏移量，同样采用压缩的方式
* redo_log_body 表示每个 redo log 的数据部分，恢复时需要调用相应的函数进行解析

![alt text](image-1.png)

从 redo log buffer 往磁盘写入时，是按 512 个 bytes，也就是一个扇区的大小进行写入。因为扇区是写入的最小单位 -> 因此可以保证写入一定是成功的 -> 因此在 redo log 的写入过程中不需要有 doublewrite。*（暂时不太理解）*

* innodb_flush_log_at_trx_commit：有效值有 0、1、2。
    0：当提交事务时，不将事务的 redo log 写入磁盘上的日志文件，而是等待主线程（master thread）每秒的刷新
    1：在执行 commit 时将 redo log buffer 同步写到磁盘，即伴有 fsync 的调用
    2：将 redo log 异步写到磁盘，即写到文件系统的缓存中 -> 因此不能完全保证在执行 commit 时肯定会写入 redo log 文件，只是有这个动作发生

    因此，为了保证 ACID 中的 D(Durability)，必须将 innodb_flush_log_at_trx_commit 设为 1，即每当有事务提交时，就必须确保事务都已经写入 redo log 文件 -> 这样当数据库宕机时可以通过 redo log 文件恢复，并保证可以恢复已经提交的事务。

    而设置为 0 或 2 都有可能发生恢复时部分事务的丢失。不同之处在于设置为 2 时，当 MySQL 数据库宕机而操作系统和服务器并没有宕机时，由于此时未写入磁盘的事务日志保存在文件系统缓存中，当恢复时同样能保证数据不丢失。

# 4 表

## 4.1 索引组织表（index organized table）

在 InnoDB 中，表都是根据主键顺序组织存放的，这种存储方式称为 index organized table。在 InnoDB 中，每张表都有一个主键（Primary Key）。

如果在创建表时没有显式地定义主键，InnoDB 会按如下方式选择/创建主键：
* 首先判断表中是否有非空的唯一索引（Unique NOT NULL）：如果有 -> 该列为主键
* 如果不符合上述条件，InnoDB 自动创建一个 6 bytes 大小的指针

当表中有多个非空唯一索引时，InnoDB 将选择**建表时第一个定义的**非空唯一索引为主键。注意：是根据定义索引的顺序，而不是建表时列的顺序。

## 4.2 InnoDB 逻辑存储结构

在 InnoDB 中，所有数据都被逻辑地存放在一个空间中，称之为 tablespace(表空间)。表空间又由 segment(段)、extent(区)、page(页) 组成。page 在一些文档中有时也称为 block(块)。

![alt text](image-2.png)

### 4.2.1 tablespace

InnoDB 逻辑结构的最高层

注意：如果启用了 innodb_file_per_file 参数，每张表的表空间内存放的只是数据、索引、插入缓冲 Bitmap 页，而其他类的数据，如回滚（undo）信息，插入缓冲索引页、系统事务信息，二次写缓冲（Double write buffer）等还是存放在原来的共享表空间内。
-> 这也同时说明了另一个问题：即使在启用了参数 innodb_file_per_file 之后，共享表空间还是会不断地增加大小。

### 4.2.2 segment

表空间由各个段组成，常见的段有数据段、索引段、回滚段等。数据段就是 B+ 树的叶子节点（Leaf node segment），索引段就是 B+ 树的非叶子节点（Non-Leaf node segment）。回滚段比较特殊，在后面章节单独介绍。

### 4.2.3 extent

区是连续页组成的空间，**在任何情况下每个区的大小都为 1MB**。为了保证区的连续性，InnoDB 一次从磁盘申请 4~5 个区。**默认情况下 InnoDB 页的大小为 16KB，即一个区中一共有 64 个连续的页。**

### 4.2.4 page

页是 InnoDB 磁盘管理的最小单位，默认每个页的大小为 16KB，可以通过参数 innodb_page_size 设置为 4K、8K、16K。

InnoDB 中常见的页类型有：
* B-tree Node (数据页)
* undo Log Page (undo 页)
* System Page (系统页)
* Transaction system Page (事务数据页)
* Insert Buffer Bitmap (插入缓冲位图页)
* Insert Buffer Free List (插入缓冲空闲列表页)
* Uncompressed BLOB Page (未压缩的二进制大对象页)
* compressed BLOB Page (压缩的二进制大对象页)

### 4.2.5 row

每个页最多允许存放 7992 行记录

## 4.3 InnoDB 行记录格式

*（TODO：补充 Page 102~182 笔记）*

# 5 索引与算法

## 5.1 InnoDB 存储引擎索引概述

InnoDB 支持以下几种常见的索引：
* B+ 树索引
* 全文索引
* 哈希索引

InnoDB 支持的哈希索引是自适应的，InnoDB 会根据表的使用情况自动为表生成哈希索引，不能人为干预是否在一张表中生成哈希索引。

B+ 树就是传统意义上的索引。
注意：B+ 树中的 B 不是代表 binary，而是代表 balance，因为 B+ 树是从最早的平衡二叉树演化而来，但是，B+ 树不是一个二叉树。

B+ 树索引并不能找到一个给定键值的具体行，它能找到的只是被查找数据所在的页。然后数据库通过把页读入到内存，再在内存中进行查找，最后找到要查找的数据。

## 5.2 数据结构与算法

### 5.2.1 二分查找法

### 5.2.2 二叉查找树与平衡二叉树（AVL 树）

演化：二叉查找树 -> 平衡二叉树 -> B 树 -> B+ 树

## 5.3 B+ 树（Page 187~

B+ 树是为磁盘或其他直接存取设备设计的一种平衡查找树。在 B+ 树中，所有记录节点都是按键值的大小顺序存放在同一层的叶子节点上，由各叶子节点指针进行连接。

### 5.3.1 B+ 树的插入操作

* B+ 树插入的三种情况：
| Leaf Page 满 | Index Page 满 | 操作                                                         |
| ------------ | ------------- | ------------------------------------------------------------ |
| No           | No            | 直接将记录插入到叶子节点                                     |
| Yes          | No            | 1. 拆分 Leaf Page；2. 将中间的节点放入到 Index Page 中；3. 小于中间节点的记录放左边；4. 大于或等于中间节点的记录放右边 |
| Yes          | Yes           | 1. 拆分 Leaf Page；2. 小于中间节点的记录放左边；3. 大于或等于中间节点的记录放右边；4. 拆分 Index Page；5. 小于中间节点的记录放左边；6. 大于中间节点的记录放右边；7. 中间节点放入上一层 Index Page |

* B+ 树提供了类似于 AVL 树的旋转（Rotation）功能 -> 减少页的拆分（split）操作
    拆分发生在 Leaf Page 已经满，但是其左右兄弟节点没有满的情况下。
    *（疑问：所以到底是按图 5-8，根据第二种情况去拆分叶子节点，还是按图 5-10，做旋转操作呢？这里讲得不够严谨。）*

B+ 树模拟：https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html

### 5.3.2 B+ 树的删除操作

B+ 树使用 **fill factor (填充因子) 来控制树的删除变化**，50% 是 fill factor 可设的最小值。

* B+ 树删除操作的三种情况：

| 叶子节点小于 fill factor | 中间节点小于 fill factor | 操作                                                         |
| ------------------------ | ------------------------ | ------------------------------------------------------------ |
| No                       | No                       | 直接将记录从叶子节点删除，如果该节点还是 Index Page 的节点，用该节点的右节点代替 |
| Yes                      | No                       | 合并叶子节点和它的兄弟节点，同时更新 Index Page              |
| Yes                      | Yes                      | 1. 合并叶子节点和它的兄弟节点；2. 更新 Index Page；3. 合并 Index Page 和它的兄弟节点 |
*（暂时不太理解）*

*（TODO：补充 Page 183~ 笔记）*