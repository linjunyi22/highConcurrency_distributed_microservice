## Mysql

[toc]

### 1.一条查询语句是如何执行

1. 逻辑架构图

   <img src="/Users/linjunyi/Desktop/github/highConcurrency_distributed_microservice/database/mysql/image/mysql 逻辑架构图.png" alt="mysql 逻辑架构图" style="zoom: 25%;" />



---



### 2.一条更新语句是如何执行

1. **redo log** 与 **Write-Ahead Logging** 技术

   > 先写日志，再写磁盘

   > 当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log 里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。

   > 当有大量更新请求写满 **redo log** 的固定大小时，会先将日志内的数据更新到磁盘后再进行其他操作
   
2. **binlog**

   <img src="/Users/linjunyi/Desktop/github/highConcurrency_distributed_microservice/database/mysql/image/update 语句执行流程.png" alt="update 语句执行流程" style="zoom:33%;" />

   * 假设当前 ID=2 的行，字段 c 的值是 0，再假设执行 update 语句过程中在写完第一个日志后，第二个日志还没有写完期间发生了 crash

     > **先写 redo log 后写 binlog**。假设在 redo log 写完，binlog 还没有写完的时候，MySQL 进程异常重启。由于我们前面说过的，redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 c 的值是 1。
     > 但是由于 binlog 没写完就 crash 了，这时候 binlog 里面就没有记录这个语句。因此，之后备份日志的时候，存起来的 binlog 里面就没有这条语句。
     > 然后你会发现，如果需要用这个 binlog 来恢复临时库的话，由于这个语句的 binlog 丢失，这个临时库就会少了这一次更新，恢复出来的这一行 c 的值就是 0，与原库的值不同。

     > **先写 binlog 后写 redo log**。如果在 binlog 写完之后 crash，由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 c 的值是 0。但是 binlog 里面已经记录了“把 c 从 0 改成 1”这个日志。所以，在之后用 binlog 来恢复的时候就多了一个事务出来，恢复出来的这一行 c 的值就是 1，与原库的值不同。

   * **两阶段提交** 保证 redo log 与 binlog 的数据一致

3. 两者区别

   * redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
   * redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
   * redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。
   * redo log 用于保证 crash-safe 能力。innodb_flush_log_at_trx_commit 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘
   * sync_binlog 这个参数设置成 1 的时候，表示每次事务的 binlog 都持久化到磁盘



---



### 3.事务隔离

1. 隔离级别

   * **读未提交** 是指，一个事务还没提交时，它做的变更就能被别的事务看到
   * **读提交** 是指，一个事务提交之后，它做的变更才会被其他事务看到。
   * **可重复读** 是指，一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。
   * **串行化**，顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

2. **可重复读** 的事务隔离实现

   todo

3. 尽量不要使用长事务

   > 长事务意味着系统里面会存在很老的事务视图。由于这些事务随时可能访问数据库里面的任何数据，所以这个事务提交之前，数据库里面它可能用到的回滚记录都必须保留，这就会导致大量占用存储空间。



---



### 4.索引一

1. InnoDB 的索引模型

   <img src="/Users/linjunyi/Desktop/github/highConcurrency_distributed_microservice/database/mysql/image/InnoDB 的索引组织结构.png" alt="InnoDB 的索引组织结构" style="zoom:33%;" />

   * 索引类型分为 **主键索引** （图上左侧）和 **非主键索引** （图上右侧）
   * **主键索引** 的叶子节点存的是整行数据。在 InnoDB 里，主键索引也被称为 **聚簇索引**（clustered index）
   * **非主键索引** 的叶子节点内容是主键的值。在 InnoDB 里，非主键索引也被称为 **二级索引**（secondary index）。
   * **基于主键索引和普通索引的查询有什么区别**
     * 如果语句是 select * from T where ID=500，即主键查询方式，则只需要搜索 ID 这棵 B+ 树；
     * 如果语句是 select * from T where k=5，即普通索引查询方式，则需要先搜索 k 索引树，得到 ID 的值为 500，再到 ID 索引树搜索一次。这个过程称为 **回表** 。

2. 索引维护

   * 由于每个非主键索引的叶子节点上都是主键的值。如果用身份证号做主键，那么每个二级索引的叶子节点占用约 20 个字节，而如果用整型做主键，则只要 4 个字节，如果是长整型（bigint）则是 8 个字节。
   * **主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小。**
   * 从性能和存储空间方面考量，自增主键往往是更合理的选择。

3. 哪些场景适合用业务字段直接做主键 ---- KV 场景

   * 只有一个索引；
   * 该索引必须是唯一索引。

