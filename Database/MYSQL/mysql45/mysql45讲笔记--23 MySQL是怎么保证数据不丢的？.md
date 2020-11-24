# 23 | MySQL是怎么保证数据不丢的？

### binlog 的写入机制

**binlog 写入逻辑**：事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。

**binlog cache 的作用是什么？**

一个事务的binlog 是不能被拆开的，因此无论这个是我有多大，也要保证一次性写入到 binlog。那再事务未提交的过程会产生日志，这时就需要一个存储未提交的 binlog，这个就是 **binlog cache**

**binlog cache 存在哪里？**

系统给 binlog cache 分配了一块内存，**每一个线程一个**，参数 binlog_cache_size 用于控制单个线程内 binlog cache 所占用内存的大小。超过这个参数规定的大小，就要暂存到磁盘。

**binglog 写入流程**

事务提交的时候，执行器把 binlog cache 里面的完整事务写入到 binlog 中，并清空 binlog cache。如图 1 ：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-23-01.png)

<center>图 1 binlog 写盘状态</center>

我们可以看到，每一个线程都有自己的 binlog cache，但是共用一份 binlog 文件。

- 图中的 write ，指的就是把日志文件写到文件系统的 page cache，并没有把数据持久化到磁盘，所有速度比较快。
- 图中的 fsync ，才是将数据持久化到磁盘的操作。一般情况下，我们认为 fsync 才占磁盘的 IOPS。

**write 和 fsync 的时机，是由参数 sync_binlog 控制的（重要）：**

1. sync_binlog=0,表示每次提交事务都只要 write，不 fsync； （不持久化）
2. sync_binlog=1,表示每次提交事务都会执行fsync ；（每一次都进行持久化）
3. sync_binlog=N(N>1) , 表示每次提交事务都 write，但累积 N 个事务后才 fsync。

**怎么设置 sync_binlog 参数**

- 如果需要数据不丢失，那就设置成 1 ，每一次提交事务，都进行持久化
- 如果像上篇文章那样，遇到了 IO 瓶颈，可以设置成一个较大的值，假设是 M ，存在的风险是，如果发生 cash ，将会有丢失 最多 M 个事务的 binlog 。
- 在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成 0，比较常见的是将其设置为 100~1000 中的某个数值

### redo log 的写入机制

事务在执行过程中，生成的 redo log 是要先写到 redo log buffer 中的，待事务提交后，再把 redo log buffer 中的数据写到redo log 文件。

**什么是 redo log buffer？**

redo log buffer 就是一块内存，用来先存 redo 日志的，即存储事务未提交过程产生的 redo log 日志。

**redo log buffer 里面的内存，是不是每次生成后都要直接持久化到磁盘呢？**

不需要。

但是，事务还未提交的时候，redo log buffer 中的部分日志有可能被持久化到磁盘中。

我们来看一下，redo log 可能存在的三种状态，如图2：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-23-02.png)

<center>图 2 MySQL redo log 存储状态</center>

1. 存在 redo log buffer 中，物理上是在 MySQL进程内存中，就是图中红色部分；
2. 写到磁盘（write），但没有持久化（fsync),物理上是在文件系统的 page cache 里面，也就是图中黄色部分；
3. 持久化到磁盘，对应的是 hard disk ，也就是图中绿色部分

日志写到 redo log buffer 是很快的，wirte 到 page cache 也差不多，但是持久化到磁盘的速度就慢多了。

**为了控制 redo log 的写入策略，InnoDB 提供了 innodb_flush_log_at_trx_commit 参数，它有三种可能取值：**

1. 设置为 0，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中 ;
2. 设置为 1 ,表示每次事务提交时都将 redo log 直接持久化到磁盘；
3. 设置为 2 ,表示每次事务提交时都只是把 redo log 写到 page cache;

其中，InnoDB 有一个后台程序，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到硬盘。

注意，因为事务执行过程中，所有的 redo log 是写在同一个 redo log buffer 中的，这些 redo log，也会被后台线程一起持久化到磁盘，也就是说，一个没有提交的事务的 redo log，也是可能已经持久化到磁盘的。

除了后台线程每秒一次的轮询操作外，还有两种可能会让一个没有提交的事务的 redo log 写入到磁盘中。

1. **redo log buffer 占用空间大的时候**，redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘。（注意，由于这个事务并没有提交，所以这个写盘动作只是 write，而没有调用 fsync，也就是**只留在了文件系统的 page cache**。）
2. **并行的事务提交的时候**，并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘，因为 多个事务都是写在 redo log buffer 中的 如果我 innodb_flush_log_at_trx_commit  设置成为 1，如果有一个事务提交，redo log buffer 里面的日志就会持久化到磁盘中。

> 这里需要说明的是，我们介绍两阶段提交的时候说过，时序上 redo log 先 prepare， 再写 binlog，最后再把 redo log commit。
>
> 如果把 innodb_flush_log_at_trx_commit 设置成 1，那么 redo log 在 prepare 阶段就要持久化一次，因为有一个崩溃恢复逻辑是要依赖于 prepare 的 redo log，再加上 binlog 来恢复的。
>
> 每秒一次后台轮询刷盘，再加上崩溃恢复这个逻辑，InnoDB 就认为 redo log 在 commit 的时候就不需要 fsync 了，只会 write 到文件系统的 page cache 中就够了。



**什么是双“ 1 "配置**

指的就是 sync_binlog 和 innodb_flush_log_at_trx_commit 都设置成 1。也就是说，一个事务完整提交前，需要等待两次刷盘，一次是 redo log（prepare 阶段），一次是 binlog。



### 组提交（group commit）机制

 日志逻辑序列号 (log sequence number,LSN): LSN 是单调递增的，用来对应 redo log 的一个个 写入点。每次写入长度为 length 的 redo log， LSN 的值就会加上 length。

如图 3 所示，是三个并发事务 (trx1, trx2, trx3) 在 prepare 阶段，都写完 redo log buffer，持久化到磁盘的过程，对应的 LSN 分别是 50、120 和 160。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-23-03.png)

<center>图 3 redo log 组提交</center>

分析一下图中：

1. trx1 是第一个到达的，会被选为这组的 leader；
2. 等 tx1 要开始写盘的时候，这个组里面已经有了三个事务，这时候 LSN 也变成 160；
3. trx1 去写盘的时候，带的就是 LSN=160，因此等 tx1 返回时，所有 LSN小于等于 160 的 redo log，都已经被持久化到磁盘；
4. 这时候 trx2 和 trx3 就可以直接返回了。

所以，一次组提交里面，组员越多，节约磁盘 IOPS 的效果越好。但如果只有单线程压测，那就只能老老实实地一个事务对应一次持久化操作了。

**应用组提交**

为了让一次 fsync 带的组员更多，MySQL 有一个很有趣的优化：拖时间。在介绍两阶段提交的时候，有一个图，现在我把它截过来。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-23-04.png)

<center>图 4 两阶段提交</center>

图中，我把“写 binlog”当成一个动作。但实际上，写 binlog 是分成两步的：

1.  先把 binlog 从 binlog cache 中写到磁盘上的 binlog 文件； 
2.  调用 fsync 持久化。 

MySQL 为了让组提交的效果更好，把 redo log 做 fsync 的时间拖到了步骤 1 之后。也就是说，上面的图变成了这样：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-23-05.png)

<center>图 5 两阶段提交细化</center>

这么一来，**binlog 也可以组提交了**。在执行图 5 中第 4 步把 binlog fsync 到磁盘时，如果有多个事务的 binlog 已经写完了，也是一起持久化的，这样也可以减少 IOPS 的消耗。

不过通常情况下第 3 步执行得会很快，所以 binlog 的 write 和 fsync 间的间隔时间短，导致能集合到一起持久化的 binlog 比较少，因此**binlog 的组提交的效果通常不如 redo log 的效果那么好**。

**提升 binlog 组提交的效果 **

可以通过设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 来实现。

1. binlog_group_commit_sync_delay 参数，表示延迟多少微秒才调用 fsync ；
2. binlog_group_commit_sync_no_delay_count 参数，表示累积多少次以后才调用 fsync。

这两个条件是或的关系，也就是说只要有一个满足条件就会调用 fsync。

所以，当 binlog_group_commit_sync_delay 设置为 0 的时候，binlog_group_commit_sync_no_delay_count 也无效了。

**为什么说 WAL 机制是减少磁盘写？**

WAL 机制主要得益于两个方面：

1. redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快；
2. 提交机制，可以大幅度降低磁盘的 IOPS 消耗。

**如果你的 MySQL 现在出现了性能瓶颈，而且瓶颈在 IO 上，可以通过哪些方法来提升性能呢？**

1.  设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 参数，减少 binlog 的写盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险 
2. 将 sync_binlog 设置为大于 1 的值（比较常见是 100~1000）。这样做的风险是，主机掉电时会丢 binlog 日志
3. 将 innodb_flush_log_at_trx_commit 设置为 2。这样做的风险是，主机掉电的时候会丢数据。

### 疑问

1. 什么是 page cache？

   答：page cache是操作系统文件系统上 ，是在内存中申请一块内存用于存储

