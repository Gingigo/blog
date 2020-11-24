# 答疑文章（一）

## 日志相关问题 

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-15-01.jpg)

<center>图 1 两阶段提交示意图</center>
我们分析一下**在两阶段提价的不同时刻，MySQL 异常重启会出现什么现象？。**

- 如果是在图中时刻A的地方，也就是 redo log 处于 prepare 阶段之后、写 binlog之前，发生崩溃（crash），由于 binlog 还没有写， redo log 也还没有提交，所以崩溃恢复的时候，这个事务会回滚。这时候，binlog 还没写，所以也不会传到备库。

- 如果是在图中时刻B的地方 发生 crash，有两种情况，需要分开判断：

  1. 如果 redo log 里面的事务完整的，也就是已经有了 commit 表示，则直接提交；

  2. 如果redo log 里面的事务只有完整的 prepare，则判断对应事务 binlog 是否存在并完整：

     a. 如果是，则提交事务；

     b. 否则，回滚事务。

### 问题 1：MySQL 怎么知道的 binlog 是完整的？

回答：一个事务的 binlog 是有完整的格式的：

-  statement 格式的 binlog，最后会有 COMMIT；
- row 格式的 binlog ，最后会有一个 XID event;

另外，在 MySQL 5.6.2 版本以后，还引入了 binlog-checksum 参数，用来验证 binlog 内容的正确性。对于 binlog 日志由于磁盘原因，可能会在日志中间出错的情况，MySQL 可以通过校验 checksum 的结果来发现。所以，MySQL 还是有办法验证事务 binlog 的完整性的。 

### 问题2：redo log 和 binlog 是怎么关联起来的？

回答：它们有一个共同的数据字段，叫 XID。崩溃恢复的时候，会按顺序扫描 redo log；

### 问题3： 处于 perpare 阶段的 redo log 加上 完整 binlog ，重启就能恢复，MySQL 为什么要这样设计

回答：因为 binlog 日志完整，发生了崩溃，此时binlog 已经写入了，之后就会被从库使用，所以必须要恢复， 主库和备库的数据就保证了一致性 。

### 问题4：redo log 一般设置多大

回答：redo log 太小的话，会导致很快就被写满，然后不得不强行刷 redo log，这样 WAL 机制的能力就发挥不出来了。

 所以，如果是现在常见的几个 TB 的磁盘的话，就不要太小气了，直接将 redo log 设置为 4 个文件、每个文件 1GB 吧。 

### 问题5：正常运行总的实例，数据写入的最终落盘，是从redo log 更新过来还是从从 buffer pool 更新过来呢？

回答： redo log 并没有记录数据页的完整数据，所以它并没有能力自己去更新磁盘数据页，也就不存在“数据最终落盘，是由 redo log 更新过去”的情况。 

1.  如果是正常运行的实例的话，数据页被修改以后，跟磁盘的数据页不一致，称为脏页。最终数据落盘，就是把内存中的数据页写盘。这个过程，甚至与 redo log 毫无关系。 
2. 在崩溃恢复场景中，InnoDB 如果判断到一个数据页可能在崩溃恢复的时候丢失了更新，就会将它读到内存，然后让 redo log 更新内存内容。更新完成后，内存页变成脏页，就回到了第一种情况的状态。 

### 问题6：什么是 redo log buffer？是先修改内存，还是先修改redo log文件？

回答： 在一个事务的更新过程中，日志是要写多次的。比如下面这个事务： 

```mysql

begin;
insert into t1 ...
insert into t2 ...
commit;
```

这个事务要往两个表中插入记录，插入数据的过程中，生成的日志都得先保存起来，但又不能在还没 commit 的时候就直接写到 redo log 文件里。

所以，redo log buffer 就是一块内存，用来先存 redo 日志的。也就是说，在执行第一个 insert 的时候，数据的内存被修改了，redo log buffer 也写入了日志。但是，真正把日志写到 redo log 文件（文件名是 ib_logfile+ 数字），是在执行 commit 语句的时候做的。

所以是先写内存->redo log buffer -> redo log

