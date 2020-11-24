# 一条SQL更新语句是如何进行的呢？

上一篇文章我们已经学习了一条查询语句需要经过 连接器、查询缓存、分析器、优化器、执行器等功能模块最后到达存储引擎，而一条更新语句的执行流程又是怎么样的呢？

> 我们的引擎默认是InnoDB引擎

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-02-01.png)

> 建表语句：
>
> ```mysql
> mysql> create table T(ID int primary key, c int);
> ```
>
> 更新语句：
>
> ```mysql
> mysql> update T set c=c+1 where ID=2;
> ```
>
> 

1. 执行语句前需要连接数据库，这是连接器的工作
2. 上篇文章我们有记到如何更新数据这查询缓存就会失效，所以执行这条语句会把更新表T在查询缓存中的结果清空。
3. 分析器通过词法分析和语法分析知道这是一条更新语句
4. 优化器更新条件确定需要用的索引
5. 执行器负责执行这条语句

上述的执行流程和查询一条数据的步骤基本相同，但是更新流程还涉及到两个重要的日志，redo log（从做日志）和binlog（归档日志）。

## 重要的日志模块 redo log

在mysql中存在一个问题，如果每一次更新操作都是要写磁盘，然后磁盘也要找到对应的那条记录，最后再更新，整个过程的IO成本、查找成本都是很高。

WAL（Writer-Ahead Logging）技术，这种技术是先写日志，再写磁盘。（这是一种数据“缓冲器”， 先写到缓存文件中，在合适的时间或者手动去触发写到磁盘），redo log就是属于这种技术

###  redo log 如何实现WAL技术

在InnoDB中 redo log 是固定大小的，比如可以配置一组4个文件，每个文件大小是1GB，那么"缓存文件"就可以记录4GB的操作。redo log 是一个环形数据结构，从头开始写，写到末尾，又开始从头循环写，如下图

![]( https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-02-02.png) 

write pos 是当前记录的位置，一边写一边后移（图中顺时针方向），写到3好文件的末尾就回到0号文件的开头。check point就是当前要擦除的位置，也是后推并且循环的，擦除的记录更新到数据文件中。

write pos 和 check point之间（绿色部分）就是可以写入的大小，用作记录新的操作更新。如果write pos 追上check point ，表示缓存文件满了，这时候就不能继续更新，得停下来把记录更新到数据文件中。

有了redo log InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称之为**crash-safe**

> redo log 是InnoDB引擎特有的日志，作用是在数据引擎层

##  重要的日志模块 binlog

binlog是msyql Server层的日志文件，称之为binlog（归档日志）

执行上面的更新语句的流程如下图：

浅绿色表示实在InnoDB内部执行，深色表示在执行器中执行

![操作](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-02-03.png)

其中最后三步是比较绕的，将redo log 的写入拆成两个步骤：prepare和commit，这就是“两阶段提交”。

## 两阶段提交

两阶段提交是为了让两份日志之间的逻辑一致，举个例子

1. **先写redo log 后写 binlog。**假如在redo写完，binlog还没写完的时候，MySQL进程异常重启。由于redo log有carsh-safe的能力所以能够恢复数据，但binlog没有记录该日志，如果后续我们用binlog同步数据等，将会丢失本次更新。

2. **先写binlog 再写redo log**。假设在写完binlog后MySQL就异常重启，redo log没有记录到该数据的日志，从而数据没有了carsh-safe的保证，数据无法恢复，就会丢失，后续我们用binlog同步数据的时候反而多了本次 操作的记录，数据对应不上。

3. **采用两阶段提交**。有三个阶段 prepare、写binlog、commit，如果在prepared期间MySQL异常重启，由于redo log数据不完整，回滚数据；如果是在写binlog之前，发生了异常重启，由于redo log 数据完整、没有commit、binlog中没有数据，回滚数据；如果异常重启是发生在commit之前，检查prepare和写binlog的日志是否完整，如果完整执行commit，否则回滚数据。

   > redo log 和 binlog 有个共同的数据字段XID,崩溃恢复的时候，会按照顺序扫描redo log：
   >
   > 1.   如果碰到既有prepare、又有commit的redo log,就直接提交 
   > 2.  如果碰到只有prepare、而没有commit的redo log,就拿着XID去binlog找对应的事务 

## mysql 如何配置redo log 和 binlog

redo log 用于保证 crash-safe 能力。innodb_flush_log_at_trx_commit 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘。这个参数我建议你设置成 1，这样可以保证 MySQL 异常重启之后数据不丢失。

 sync_binlog 这个参数设置成 1 的时候，表示每次事务的 binlog 都持久化到磁盘。这个参数我也建议你设置成 1，这样可以保证 MySQL 异常重启之后 binlog 不丢失。 

```mysql
# 查询innodb_flush_log_at_trx_commit的值
mysql> show global variables like '%innodb_flush_log_at_trx_commit%';
# 设置innodb_flush_log_at_trx_commit的值
mysql> set global innodb_flush_log_at_trx_commit=1;
```

> 当**innodb_flush_log_at_trx_commit**的值为**0**
>
> log buffer将每一秒地写入到log file中，并且log file的flush到磁盘操作同时进行（每秒读，同步刷）
>
> 当**innodb_flush_log_at_trx_commit**的值为**1**（默认值）
>
> 每次事务提交时MySQL都会把log buffer的数据写入到log file，并且同步flush到磁盘中（每次提交，同步刷）
>
> 当**innodb_flush_log_at_trx_commit**的值为**2**
>
> 每次事务提交时MySQL都会把log buffer的数据写入到log file,但是MySQL会每秒执行一次flush（每次提交，每秒刷）
>
> 所以设置值为1时满足ACID的，当MySQL异常重启，0和2都有可能会出现一秒的输出错误，不满足ACID。

```mysql
# 查询sync_binlog配置
mysql> show global variables like '%sync_binlog%'
# 修改sync_binlog配置
mysql> set global sync_binlog=2;
```

>当**sync_binlog**的值为**0**（默认值）
>
>表示MySQL不控制binlog的刷新，由文件系统自己控制它的缓存刷新。这时性能时最好的，但是存在风险，一旦系统crash,在binlog_cache中的binlog将会丢失。
>
>当**sync_binlog**的值为**>0**，假设该值为**n**
>
>表示n次事务提交后，MySQL调用文件系统刷新操作将缓存刷到磁盘中。其中最安全的是n=1的时候（*发生crash时有可能丢失一个binlog*），每次事务提交，就刷新binlog入磁盘，但是对IO负载较大。

## 自问自答

1. **如何修改redo log的大小？**

   答：

   1. 执行`set global innodb_fast_shutdown=0 `,把所有脏数据写入磁盘；
   2. 执行`mysqladmin shutdown`,关闭数据库；
   3. 修改my.cnf文件innodb_log_file_size参数值；
   4. 执行mv ib_logfile*bak命令，将redo log移到其他目录；(MySQL5.7之后可忽略)
   5. 执行`mysql_safe --defaults-file=/etc/my.cnf --user=mysql &`，启动MySQL。

2. **为什么有两份日志**

   答：因为MySQL一开始是没有InnoDB引擎的，MySQL自带的引擎是MyISAM，当时MyISAM没有carsh-safe的能力，binlog日志只能用于归档，记录Server的操作，即是MySQL自身的归档日志，而redo log是InnoDB独有的日志，用来实现carsh-safe。

3. **两种日志有哪些不同点？**

   答：

   1. redo log 是InnoDB引擎特有的；binlog是MySQL的Server层实现，所有的引擎都可以使用

   2. redo log是物理日志，记录的是“在某个数据页做了什么修改”；binlog是逻辑日志，记录的是这个语句原始的逻辑

   > 一 个a操作，初始值 x=0 第一步 x = x+1,第二步 x=x+2。物理日志会记录 操作a： x=0  => x=3；而逻辑日志会记录 操作a 初始值x=0，x=x+1；x=x+2；

   3. redo log是循环写会，空间固定会用完；binlog 是可以追加写。

      > 追加写是指binlog文件写到一定大小之后会切换到下一个，不会覆盖以前的日志
