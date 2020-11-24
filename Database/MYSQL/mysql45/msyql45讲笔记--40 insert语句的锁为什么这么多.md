# 40 | insert语句的锁为什么这么多？

介绍了几种特殊情况下的 insert 语句 。

### insert ... select 语句

```mysql

CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(null, 1,1);
insert into t values(null, 2,2);
insert into t values(null, 3,3);
insert into t values(null, 4,4);

create table t2 like t
```

**条件：**

- 可重复读隔离级别
- binlog_format = statement
- 执行语句 `insert into t2(c,d) select c,d from t`

**问题：**

- 为什么需要给表 t 的所有行加间隙锁？

**原因：**

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-40-01.png)

<center>图 1 并发 insert 场景</center>
实际的执行效果是 ，如果 session B 先执行，由于这个语句对表 t 主键索引加了 (-∞,1]这个 next-key lock，会在语句执行完成后，才允许 session A 的 insert 语句执行。

反证法：

但如果没有锁的话，就可能出现 session B 的 insert 语句先执行，但是后写入 binlog 的情况。于是，在 binlog_format=statement 的情况下，binlog 里面就记录了这样的语句序列： 

```mysql
insert into t values(-1,-1,-1);
insert into t2(c,d) select c,d from t;
```

这个语句到了备库执行，就会把 id=-1 这一行也写到表 t2 中，出现主备不一致。

**结论：**

<font color='orange'>insert … select 是很常见的在两个表之间拷贝数据的方法。你需要注意，在可重复读隔离级别下，这个语句会给 select 的表里扫描到的记录和间隙加读锁。 </font>

### insert 循环写入

 那么，如果我们是要把这样的一行数据插入到表 t 中的话： 

```mysql
insert into t(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
```

**问题：**

- 语句执行流程是怎么样的？
- 扫描行数又是多少？

**分析：**

1. 查看慢查询日志:

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-40-02.png)

   可以看到这个时候的 Rows_examined 的值是5。

2. 查看explan

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-40-03.png)

   可以看出“Using temporary” 使用了临时表，就是把 t 表中的数据读到临时表中。

3. 查看 InnoDB 扫描了多少行

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-40-04.png)

   这个语句执行前后，Innodb_rows_read 的值增加了 4。因为默认临时表是使用 Memory 引擎的，所以这 4 行查的都是表 t，也就是说对表 t 做了全表扫描

**解答：**

这样，我们就把整个执行过程理清楚了： 

1.  创建临时表，表里有两个字段 c 和 d。 
2.  按照索引 c 扫描表 t，依次取 c=4、3、2、1，然后回表，读到 c 和 d 的值写入临时表。这时，Rows_examined=4。 
3. 由于语义里面有 limit 1，所以只取了临时表的第一行，再插入到表 t 中。这时，Rows_examined 的值加 1，变成了 5。

**结论：**

也就是说，这个语句会导致在表 t 上做全表扫描，并且会给索引 c 上的所有间隙都加上共享的 next-key lock。所以，这个语句执行期间，其他事务不能在这个表上插入数据。 

至于这个语句的执行为什么需要临时表，原因是这类一边遍历数据，一边更新数据的情况，如果读出来的数据直接写回原表，就可能在遍历过程中，读到刚刚插入的记录，新插入的记录如果参与计算逻辑，就跟语义不符。

**产生新的问题**：

-  由于实现上这个语句没有在子查询中就直接使用 limit 1，从而导致了这个语句的执行需要遍历整个表 t 。如何进行优化？

**最终结论**

<font color='orange'>而如果 insert 和 select 的对象是同一个表，则有可能会造成循环写入。这种情况下，我们需要引入用户临时表来做优化。</font>

**解决新问题：**

由于涉及的语句的数据很小，用内存临时表来做这个优化：

```mysql
create temporary table temp_t(c int,d int) engine=memory;
insert into temp_t  (select c+1, d from t force index(c) order by c desc limit 1);
insert into t select * from temp_t;
drop table temp_t;
```

###  insert 唯一键冲突 

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-40-05.png)

<center>图2 唯一键冲突加锁 </center>
**条件：**

- innoDB 引擎的可重复读隔离级别下

**现象：**

也就是说，session A 执行的 insert 语句，**发生唯一键冲突的时候，并不只是简单地报错返回，还在冲突的索引上加了锁**。我们前面说过，一个 next-key lock 就是由它右边界的值定义的。这时候，session A 持有索引 c 上的 (5,10]共享 next-key lock（读锁）

**结论**

<font color='orange'> insert 语句如果出现唯一键冲突，会在冲突的唯一值上加**共享的 next-key lock(S 锁)**。因此，碰到由于唯一键约束导致报错后，要尽快提交或回滚事务，避免加锁时间过长。 </font>

**经典场景**

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-40-06.png)

<center> 图 3 唯一键冲突 -- 死锁 </center>
在 session A 执行 rollback 语句回滚的时候，session C 几乎同时发现死锁并返回。

这个死锁产生的逻辑是这样的： 

1. 在 T1 时刻，启动 session A，并执行 insert 语句，此时在索引 c 的 c=5 上加了记录锁。注意，这个索引是唯一索引，因此退化为记录锁 。
2.  在 T2 时刻，session B 要执行相同的 insert 语句，发现了唯一键冲突，加上读锁；同样地，session C 也在索引 c 上，c=5 这一个记录上，加了读锁。 
3.  T3 时刻，session A 回滚。这时候，session B 和 session C 都试图继续执行插入操作，都要加上写锁。两个 session 都要等待对方的行锁，所以就出现了死锁。 

### insert into … on duplicate key update

**概念**

insert into … on duplicate key update 这个语义的逻辑是，插入一行数据，如果碰到**唯一键约束**，就执行后面的更新语句。

**特殊性**

注意，如果有多个列违反了唯一性约束，就会按照索引的顺序，修改跟第一个索引冲突的行。

例子：

假设表 t 中里面已经又了 （1，1，1）和（2，2，2）这两行，我们看一下执行下面语句的效果

```mysql
insert into t values(2,1,100) on duplicate key update d=100
```

结果如下：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-40-07.png)

<center>图4 两个唯一键同时冲突 </center>
可以看到，主键 id 是先判断的，MySQL 认为这个语句跟 id=2 这一行冲突，所以修改的是 id=2 的行。

需要注意的是，执行这条语句的 affected rows 返回的是 2，很容易造成误解。实际上，真正更新的只有一行，只是在代码实现上，insert 和 update 都认为自己成功了，update 计数加了 1， insert 计数也加了 1。
