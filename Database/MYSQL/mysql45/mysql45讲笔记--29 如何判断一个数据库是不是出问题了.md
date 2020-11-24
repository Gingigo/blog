# 如何判断一个数据库是不是出问题了？

##  外部检测 

**为什么要检测数据是否有问题？**

因为，当数据库出现并发查询（cpu占用高）过高或者IO读写高等问题时，如果时双M架构的，就要采取主备切换策略，防止出现业务崩溃问题。

### select 1 判断

**核心原理：** 如果执行 select 1能返回 1 就证明能用。

实际上，select 1 成功返回，只能说明这个库的进程还在， 并不能说明主库没问题 。

看一种场景

```mysql

set global innodb_thread_concurrency=3; # 控制 InnoDB 的并发线程上限为 3

CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

 insert into t values(1,1)
```

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-29-01.png)

<center>图 1 查询 blocked</center>

session D 里面，select 1 是能执行成功的，但是查询表 t 的语句会被堵住 。也就是说，如果这时候我们用 select 1 来检测实例是否正常的话，是检测不出问题的。

其中，在 InnoDB 中，innodb_thread_concurrency 这个参数的默认值是 0，表示不限制并发线程数量。建议把 innodb_thread_concurrency 设置为 64~128 之间的值。

**并发连接和并发查询 ？**

并发连接和并发查询，并不是同一个概念。你在 show processlist 的结果里，看到的几千个连接，指的就是并发连接。而“当前正在执行”的语句，才是我们所说的并发查询。

**那进入锁等待的并发查询很多，128够吗？**

实际上，**在线程进入锁等待以后，并发线程的计数会减一**，也就是说等行锁（也包括间隙锁）的线程是不算在 128 里面的。

###  查表判断 

上面的例子已经证明了 select 1 的不足，我们改进一下。 为了能够检测 InnoDB 并发线程数过多导致的系统不可用情况，我们需要找一个InnoDB 的场景，查一下。

**核心原理：**一般的做法是，在系统库（mysql 库）里创建一个表，比如命名为 health_check，里面只放一行数据，然后定期执行

```mysql
mysql> select * from mysql.health_check; 
```

使用这个方法，我们可以检测出由于并发线程过多导致的数据库不可用的情况。

**解决一个问题，还有一个问题？**

这样我们解决了CPU高的问题，还有一个问题，那就是“磁盘的空间占用率高” 也是数据库不正常的表现。

比如 ，更新事务要写 binlog，而一旦 binlog 所在磁盘的空间占用率达到 100%，那么所有的更新语句和事务提交的 commit 语句就都会被堵住。但是，系统这时候还是可以正常读数据的

因此，我们还是把这条监控语句再改进一下。接下来，我们就看看把查询语句改成更新语句后的效果。

### 更新判断

**核心原理：**（查询更新都检测）既然要更新，就要找个有意义的字段，常见做法是放一个 timestamp 字段，用来表示最后一次执行检测的时间。这条更新语句类似于：

```mysql
mysql> update mysql.health_check set t_modified=now();
```

节点可用性的检测都应该包含主库和备库。如果用更新来检测主库的话，那么备库也要进行更新检测。

**主备同时更新一行可能会有问题？**

备库的检测也是要写 binlog 的。由于我们一般会把数据库 A 和 B 的主备关系设计为双 M 结构，所以在备库 B 上执行的检测命令，也要发回给主库 A。

但是，如果主库 A 和备库 B 都用相同的更新命令，就可能出现行冲突，也就是可能会导致主备同步停止。所以，现在看来 mysql.health_check 这个表就不能只有一行数据了。

**主备库更新判断升级版**

为了让主备之间的更新不产生冲突，我们可以在 mysql.health_check 表上存入多行数据，并用 A、B 的 server_id 做主键。 

```mysql

mysql> CREATE TABLE `health_check` (
  `id` int(11) NOT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

/* 检测命令 */
insert into mysql.health_check(id, t_modified) values (@@server_id, now()) on duplicate key update t_modified=now();
```

由于 MySQL 规定了主库和备库的 server_id 必须不同（否则创建主备关系的时候就会报错），这样就可以保证主、备库各自的检测命令不会发生冲突。

###  "判定慢"问题？

 **更新语句，如果失败或者超时，就可以发起主备切换了，为什么还会有判定慢的问题呢？**

其实，这里涉及的时服务器IO资源分配的问题。

首先，所有的检测逻辑都需要一个超时时间 N。执行一条 update 语句，超过 N 秒后还不返回，就认为系统不可用。

你可以设想一个日志盘的 IO 利用率已经是 100% 的场景。这时候，整个系统响应非常慢，已经需要做主备切换了。

但是你要知道，IO 利用率 100% 表示系统的 IO 是在工作的，**每个请求都有机会获得 IO 资源，执行自己的任务**。而我们的检测使用的 update 命令，需要的资源很少，所以可能在拿到 IO 资源的时候就可以提交成功，并且在超时时间 N 秒未到达之前就返回给了检测系统。 

检测系统一看，update 命令没有超时，于是就得到了“系统正常”的结论。

**根本原因**

我们上面说的所有方法，都是基于外部检测的。外部检测天然有一个问题，就是随机性。

因为，外部检测都需要定时轮询，所以系统可能已经出问题了，但是却需要等到下一个检测发起执行语句的时候，我们才有可能发现问题。而且，如果你的运气不够好的话，可能第一次轮询还不能发现，这就会导致切换慢的问题

## 内部统计

**针对磁盘利用率这个问题**，如果 MySQL 可以告诉我们，内部每一次 IO 请求的时间，那我们判断数据库是否出问题的方法就可靠得多了。

其实，MySQL 5.6 版本以后提供的 performance_schema 库，就在 file_summary_by_event_name 表里统计了每次 IO 请求的时间(包括redo log 和 binlog 的统计数据)。

> 如果打开所有的 performance_schema 项，性能大概会下降 10% 左右。所以，我建议你只打开自己需要的项进行统计

**如何利用file_summary_by_event_name来做判断呢?**

很简单，我们可以通过查出 binlog 的最大 IO 写入的时间，设置一个阈值，如果超出找个值，就判断 磁盘利用率高。 取到你需要的信息 ，把之前的统计信息清空。这样如果后面的监控中，再次出现这个异常，就可以加入监控累积值了。（当然，根据实际情况设置策略，不清楚看原文，查文档）

