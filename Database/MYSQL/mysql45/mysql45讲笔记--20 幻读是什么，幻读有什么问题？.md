# 20 | 幻读是什么，幻读有什么问题？

举个例子，建表和初始化语句如下：

```msyql

CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

**问题**

下面的语句序列，是怎么加锁的，加的锁又是什么时候释放的呢？

```mysql
-- session 1:
begin;
select * from t where d=5 for update;
commit;
```

看一下在可重复读级别下,session 1 未提交，对 session 2 的执行情况

```mysql
-- session 2
-- REPEATABLE-READ
select @@transaction_isolation;
-- 可以查看
select * from t;
-- 不可以更新
update t set c=1 where id=0;
-- 不可以删除
delete from t where id= 0;
-- 不可以新增
insert into t values(30,30,30)
```

实验结果是 **在可重复读级别下，全表加了锁，只能看不能写**。

看一下在读提交隔离级别下,session 1 未提交，对 session 3 的执行情况

```mysql
-- session 3
-- READ-COMMITTED
select @@transaction_isolation;
-- 可以查看
select * from t;
-- 可以更新
update t set c=1 where id=0;
-- 可以删除
delete from t where id= 0;
-- 可以新增
insert into t values(30,30,30)
```

实验结果是 **在读提交隔离级别，没有加锁，读写正常**。

比较好理解的是，这个语句回命中 d=5 的这一行，对应的主键 id=5，因此在 select 执行完成后，id 这一行回加一个写锁，而由于两阶段协议，这个写锁回在执行 commit 语句的时候释放。

>  **先不要看这里，看完全文再来看这个问题**
>
> 这里我做了一个小实验, 把加锁的SQL 语句 where 条件中的  d 改成 了 c。
>
> ``` msyql
> -- session 4:
> begin;
> select * from t where c=5 for update;
> commit;
> ```
>
> 在可重复读隔离级别下，session 4 未提交，对 session 5 的执行情况
>
> ```msyql
> -- session 5
> -- REPEATABLE-READ
> select @@transaction_isolation;
> -- 可以查看
> select * from t;
> -- 不可以更新
> update t set c=1 where id=25;
> update t set c=6 where id=25;
> update t set c=10 where id=25;
> -- 可以更新
> update t set c=11 where id=25;
> 
> -- 不可以新增
> insert into t values(1,1,1)
> insert into t values(6,6,6)
> -- 可以新增
> insert into t values(11,11,11)
> 
> -- 可以删除
> delete from t where id= 0;
> delete from t where id= 15;
> -- 不可以删除
> delete from t where id= 5;
> ```
>
> 下一篇文章 有答案
>

<font color='orange'>**由于字段 d 上没有索引，因此这条查询语句会做全表扫描**</font>，那么，其他被扫描到的，但是不满足条件的 5 行记录上，会不会被加锁呢？

我们知道，InnoDB 的默认事务隔离级别是可重复读，<font color='orange'>所以本文接下来没有特殊说明的部分，都是设定在可重复读隔离级别下。</font>

## 幻读是什么

还是以上面问题展开，我们来分析一下，如果只在 id=5这一行加锁，而其他行不加锁的话，会怎么样。

下面先来看一下这个场景（ 注意：<font color='orange'>这是我假设的一个场景</font> ）：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-20-01.png)

<center>图 1 假设只在 id=5 这一行加行锁</center>
可以看到，session A 里执行了三次查询，分别是Q1、Q2 和 Q3。它们的 SQL 语句相同，都是 select * from t where d=5 for update。这个语句的意思是，查所有 d=5 行，而且使用的是当前读，并且加上写锁，现在我们看一下这三条 SQL 语句，分别会返回什么结果。

1. Q1 只返回 id=5 这一行；
2. 在 T2 时刻，session B 把 id=0 这一行的 d 值改成了5，因此 T3 时刻查出来的是 id=0和 id=5 这两行；
3. 在 T4 时刻，session C 又插入一行（1，1，5），此时 T5 时刻 Q3 查出来的是 id=0、id=1 和 id=5 这三行。

其中，Q3 读到 id=1 这一行的现象，被称为“幻读”。也就是说,<font color='orange'>幻读是指一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行</font>。

这里对“幻读” 做一个说明：

1. 在可重复读隔离级别下，普通的查询就是快照读，是不会看到别的事务插入的数据的。因此，**幻读在“当前读”下才会出现**。（备注，读提交隔离级别也是会出现幻读）
2. 上面 session B 的修改结果，被 session A 之后的 select 语句用“当前读”看到，不能成为幻读。**幻读仅专指“新插入的行”**（备注，“新插入/删除”也可以称之为幻读，但修改不行）

因为这三个查询都是加了 for update，都是当前读。而当前读的规则，就是要能读到所有已经提交的记录的最新值。并且，session B 和 sessionC 的两条语句，执行后就会提交，所以 Q2 和 Q3 就是应该看到这两个事务的操作效果，而且也看到了，这跟事务的可见性规则并不矛盾。

但是，这是不是真的没问题呢？

 不，这里还真就有问题。 （哈哈，作者是用了反证法）

### 幻读有什么问题

**首先是语义上**，session A 在 T1 时刻就声明了，“我要把所有 d=5 得行锁住，不准别的事务进行读写操作”。而实际上，这个语义被破坏了。

如果现在这样看感觉还不明显的话，我再往 session B 和 session C 里面分别加一条 SQL 语句，你再看看会出现什么现象。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-20-02.png)

<center>图 2 假设只在 id=5 这一行加行锁 -- 语义被破坏</center>
session B 的第二条语句 update t set c=5 where id=0，语义是“我把 id=0、d=5 这一行的 c 值，改成了 5”。

由于在 T1 时刻，session A 还只是给 id=5 这一行加了行锁， 并没有给 id=0 这行加上锁。因此，session B 在 T2 时刻，是可以执行这两条 update 语句的。这样，就破坏了 session A 里 Q1 语句要锁住所有 d=5 的行的加锁声明。

session C 也是一样的道理，对 id=1 这一行的修改，也是破坏了 Q1 的加锁声明。

**其次，是数据一致性的问题。**

我们知道，<font color='orange'>锁得设计是为了保证数据一致性。而这个一致性，不止是数据库内部数据状态在此刻得一致性，含包含了数据和日志在逻辑上得一致性。</font>

为了说明这个问题，session A 在 T1 时刻再加上一个更新语句，即 ：update  t set d=100 where d=5。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-20-03.png)

<center>图 3 假设只在 id=5 这一行加行锁 -- 数据一致性问题</center>
update 的加锁语义和 select …for update 是一致的，所以这时候加上这条 update 语句也很合理。session A 声明说“要给 d=5 的语句加上锁”，就是为了要更新数据，新加的这条 update 语句就是把它认为加上了锁的这一行的 d 值修改成了 100。

 现在，我们来分析一下图 3 执行完成后，数据库里会是什么结果。 

1. 经过 T1 时刻，id=5 这一行变成了（5，5，100），当然这个结果最终是在 T6 时刻正式提交得；
2. 经过 T2 时刻，id=0 这一行表成了（0，5，5）；
3. 经过 T4 时刻，表里面多了一行（1，5，5）；
4. 其他行跟这个执行序列无关，保持不变。

 这样看，这些数据也没啥问题，但是我们再来看看这时候 binlog 里面的内容。 

1.  T2 时刻，session B 事务提交，写入了两条语句； 
2.  T4 时刻，session C 事务提交，写入了两条语句； 
3.  T6 时刻，session A 事务提交，写入了 update t set d=100 where d=5 这条语句。 

 我统一放到一起的话，就是这样的： 

```mysql
update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/

insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

update t set d=100 where d=5;/*所有d=5的行，d改成100*/
```

好，你应该看出问题了。这个语句序列，不论是拿到备库去执行，还是以后用 binlog 来克隆一个库，这三行的结果，都变成了 (0,5,100)、(1,5,100) 和 (5,5,100)。

也就是说，id=0 和 id=1 这两行，发生了数据不一致。这个问题很严重，是不行的。

到这里，我们再回顾一下，这个数据不一致到底是怎么引入的？

我们分析一下可以知道，这是我们假设“select * from t where d=5 for update 这条语句只给 d=5 这一行，也就是 id=5 的这一行加锁”导致的。

所以我们认为，上面的设定不合理，要改。

那怎么改呢？我们把扫描过程中碰到的行，也都加上写锁，再来看看执行效果。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-20-04.png)

<center>图 4 假设扫描到的行都被加上了行锁</center>
由于 session A 把所有的行都加了写锁，所以 session B 在执行第一个 update 语句的时候就被锁住了。需要等到 T6 时刻 session A 提交以后，session B 才能继续执行。

这样对于 id=0 这一行，在数据库里的最终结果还是 (0,5,5)。在 binlog 里面，执行序列是这样的：

```mysql
insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

update t set d=100 where d=5;/*所有d=5的行，d改成100*/

update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/
```

可以看到，按照日志顺序执行，id=0 这一行的最终结果也是 (0,5,5)。所以，id=0 这一行的问题解决了。

但同时你也可以看到，id=1 这一行，在数据库里面的结果是 (1,5,5)，而根据 binlog 的执行结果是 (1,5,100)，也就是说幻读的问题还是没有解决。为什么我们已经这么“凶残”地，把所有的记录都上了锁，还是阻止不了 id=1 这一行的插入和更新呢？

原因很简单。在 T3 时刻，我们给所有行加锁的时候，**id=1 这一行还不存在，不存在也就加不上锁。**

也就是说，即使**把所有的记录都加上锁，还是阻止不了新插入的记录**，这也是为什么“幻读”会被单独拿出来解决的原因。

接下来，我们再看看 InnoDB 怎么解决幻读的问题。

## 如何解决幻读？

现在我们知道了，产生幻读得原因是，行锁只能锁住行，但是新插入记录这个动作，要更新得是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB 只好引进新的锁，也就是间隙锁（Gap Lock）。

顾名思义，间隙锁，锁的就是两个值之间的空隙。比如文章开头的表 t，初始化插入了 6 个记录，这就产生了 7 个间隙。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-20-05.png)

<center>图 5 表 t 主键索引上的行锁和间隙锁</center>
这样，当你执行 select * from t where d=5 for update 的时候，就不止是给数据库中已有的 6 个记录加上行锁，同时加上了 7个间隙锁。这样就确保了无法插入新的记录。

也就是说这时候，**在一行行扫描的过程中，不仅将给行加上了锁，还给两边的间隙，加上了间隙锁。**

现在我们知道了，数据行是可以加上锁的实体，数据行之间的间隙，也是可以加上锁的实体。但是间隙锁跟我们之前碰到过的锁都不太一样。 

比如行锁， 分成读锁和写锁 ，有冲突关系的是“另外一个行锁” ，如读锁与写锁，写锁与写锁 之间都是冲突关系。

但是间隙锁不太一样，**跟间隙锁存在冲突关系的是，“往这个间隙中间插入一个记录” 这个操作**。间隙锁之间都不存在冲突关系。

这句话不太好理解，举个例子：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-20-06.png)

<center>图 6 间隙锁之间不互锁</center>
这里 session B 并不会别堵住。因为表 t 里面并没有 c=7 这记录，因此 session A 加的是（5，10）。而 session B 也是在这个间隙加的间隙锁。它们有共同目标，即保护这个间隙，不允许插入值。但，它们之间是不冲突的。

> 这里需要注意的是 c=7 是不存在的，所以 图6 才不会读锁和写锁冲突。如果把 c=7 换成 c=5 会先进入读锁和写锁冲突。

**间隙锁和行锁合称 next-key lock**，每个 next-key lock 是前开后闭。也就是说，我们的表 t 初始化以后，如果用 select * from  t for update 要把整个表所有记录锁起来，就形成 7 个 next-key lock，分别是 (-∞,0]、(0,5]、(5,10]、(10,15]、(15,20]、(20, 25]、(25, +supremum]。

你可能会问说，这个 supremum 从哪儿来的呢？

这是因为 +∞是开区间。实现上，InnoDB 给每个索引加了一个不存在的最大值 supremum，这样才符合我们前面说的“都是前开后闭区间”。

## 解决幻读，也产生新的问题

**间隙锁和 next-key lock 的引入，帮我们解决了幻读的问题，但同时也带来了一些“困扰”。**

举个例子：，业务逻辑这样的：任意锁住一行，如果这一行不存在的话就插入，如果存在这一行就更新它的数据，代码如下：

```mysql
begin;
select * from t where id=N for update;

/*如果行不存在*/
insert into t values(N,N,N);
/*如果行存在*/
update t set d=N set id=N;

commit;
```

可能你会说，这个不是 insert … on duplicate key update 就能解决吗？但其实在有多个唯一键的时候，这个方法是不能满足这位提问同学的需求的。至于为什么，我会在后面的文章中再展开说明。

> 作者说以后说，我就忍不住自己查了，看一个例子就知道为什么了：
>
> ```mysql
> mysql> create table test(
> 	id int not null primary key,
> 	num int not null UNIQUE key,
> 	tid int not null);
> Query OK, 0 rows affected (0.02 sec)
> 
> mysql> insert into test values(1,1,1), (2,2,2);
> Query OK, 2 rows affected (0.01 sec)
> Records: 2  Duplicates: 0  Warnings: 0
> 
> mysql> select * from test;
> +----+-----+-----+
> | id | num | tid |
> +----+-----+-----+
> |  1 |   1 |   1 |
> |  2 |   2 |   2 |
> +----+-----+-----+
> 2 rows in set (0.03 sec)
> 
> mysql> insert into test values(1,2,3) on duplicate key update tid = tid + 1;
> Query OK, 2 rows affected (0.00 sec)
> 
> mysql> select * from test;
> +----+-----+-----+
> | id | num | tid |
> +----+-----+-----+
> |  1 |   1 |   2 |
> |  2 |   2 |   2 |
> +----+-----+-----+
> 2 rows in set (0.03 sec)
> ```
>
> 插入的数据在两条记录上产生了冲突，然而执行后只有第一条记录被修改。

这个逻辑一旦有并发，就会碰到死锁。你一定也觉得奇怪，这个逻辑每次操作前用 for update 锁起来，已经是最严格的模式了，怎么还会出现死锁？

这里，我用两个 session 来模拟并发，并假设 N=9。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-20-07.png)

<center>图 7 间隙锁导致的死锁</center>
你看到了，其实都不需要用到后面的 update 语句，就已经形成死锁了。我们按语句执行顺序来分析一下：

1. seesion A 执行 select ... for update 语句，由于 id=9 这一行并不存在，因此会加上间隙锁（5，10） ；
2. session B 执行 select ... for update 语句，同样会加上间隙锁 （5，10），间隙锁之间不会冲突， 因此这个语句可以执行成功；
3. session B 试图插入一行 (9,9,9)，被 session A 的间隙锁挡住了，只好进入等待；
4. session A 试图插入一行 (9,9,9)，被 session B 的间隙锁挡住了。

至此，两个 session 进入互相等待状态，形成死锁。当然，InnoDB 的死锁检测马上就发现了这对死锁关系，让 session A 的 insert 语句报错返回了。

你现在知道了，**间隙锁的引入，可能会导致同样的语句锁住更大的范围，这其实是影响了并发度的 **。

我在文章一开始就说过，如果没有特别说明，今天和你分析的问题都是在可重复读隔离级别下的，**间隙锁是在可重复读隔离级别下才会生效的**

所以，你如果**把隔离级别设置为读提交**的话，就没有间隙锁了。但同时，你要解决可能出现的数据和日志不一致问题，**需要把 binlog 格式设置为 row**。这，也是现在不少公司使用的配置组合。
