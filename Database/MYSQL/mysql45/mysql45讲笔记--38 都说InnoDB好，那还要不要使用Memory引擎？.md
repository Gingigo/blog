# 38 | 都说InnoDB好，那还要不要使用Memory引擎？

### 内存表的数据组织结构

```mysql
create table t(id int primary key,c int) engine=Memory;
insert into t values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6),(7,7),(8,8),(9,9),(0,0);
```

以下是 t 表的结构图：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-38-01.png)

<center>图 1 内存表 t 结构图</center>

可以看出，内存表的数据部分以数组方式单独存放，而主键 id 索引里，存的是每个数据的位置。主键 id 是 hash 索引，可以看到索引上的 key 并不是有序的。

> 如果执行 select * 并不会走索引，会直接扫描数据数组。

**InnoDB 和 Memory 引擎的数组组织方式：**

- InnoDB 引擎把数据放在主键索引上，其他索引上保存的是主键id。这个方式，称之为**索引组织表**（Index Organizied Table）
- Memory 引擎采用的是把数据单独存放到数组，索引上保存数据的位置的数据组织方式。称之为**堆组织表**（Heap Organizied Table）

**InnODB 和 Memory 引擎的对比：**

1. InnoDB 表的数据总是有序存放，而 Memory 表就是按照写入的顺序存放；
2. 当数据文件有空洞的时候 InnoDB 表插入新数据的时候，为了保证数据的有序性，只能在固定的位置写入新值，而 Memory 表找到空位就可以插入新值。
3. 数据位置发生变化的时候，InnoDB 表只需要修改主键索引，而 Memory 表需要修改所有的索引。
4. InnoDB 表用主键索引查询的时只需要走一次索引查找，用普通索引查找的时候，需要走两次索引查找（覆盖索引除外）。而 Memory 表没有区别，所有索引都是平等的。
5. InnoDB 支持长数据类型，不同记录长度可能不同；Memory 表不支持 Blob 和 Text 字段，并且即使定义了 varchar(N)，实际也当作 char(N)，也就是固定长度字符串来存储，因此内存表的每行数据长度相同。

### hash 索引和 B-Tree 索引

**内存表也是支持 B-Tree 索引**

```mysql
alter table t add index a_btree_index using btree (id)
```

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-38-02.png)

<center>图 2 表t的数据表组织 -- 增加 B-Tree 索引</center>

### 内存表劣势

**内存表的优势和原因**

- 优势 ：
  - 速度快
- 原因：
  - Memory 引擎支持 hash 索引
  - 主要原因是数据都存在内存中

**锁粒度问题**

内存表不支持行锁，只支持表锁，因此，一张表只要更新，就会堵住其他所有在这个表上的**读写操作**。

和行锁比起来，表锁对并发访问的支持不够好。所以，内存表锁粒度问题，决定了它在处理并发事务的时候，性能也不会太好。

**数据持久化问题**

Memory 引擎的表，在数据库重启的时候，所有的内存表都会被清空。

还有有一个有趣特性，由于 MySQL 知道重启之后，内存表的数据丢失。所以，担心主库重启之后，出现主备不一致，MySQL 在实现上做了一件事：在数据库重启之后，往 binlog 里面写了一行 delete from t。

如果是双 M 结构，无论主备重启数据，Memory 引擎的表都会被清空。

**内存表适不适合在生产库作为普通数据表使用？**

先说答案:**内存表并不适合在生产环境上作为普通数据表使用 **

你可能会想，但是，内存表它执行速度快啊！

衡量指标：

1. 如果你表更新量大，那么并发度是一个很重要的参考指标，InnoDB 支持行锁，并发度比内存表好
2. 能放到内存表的数量都不大。如果你考虑的是读的性能，一个读 QPS 很高并且数据量不大的表，即使是使用 InnoDB，数据也是都会缓存在 InnoDB Buffer Pool 里的。因此，使用 InnoDB 表的读性能也不会差。

所以,**建议你把普通内存表都用 InnoDB 表来代替 **

### Memory 引擎表应用

存在即合理，在 join 语句的时候可以，看情形用 memory 引擎的临时表来优化join。

内存临时表刚好可以无视内存表的两个不足，主要是下面的三个原因： 

1. 临时表不会被其他线程访问，没有并发性的问题； 
2. 临时表重启后也是需要删除的，清空数据这个问题不存在； 
3. 备库的临时表也不会影响主库的用户线程。

现在，我们回过头再看一下第 35 篇 join 语句优化的例子，当时我建议的是创建一个 InnoDB 临时表，使用的语句序列是：

```mysql
create temporary table temp_t(id int primary key, a int, b int, index(b))engine=innodb;
insert into temp_t select * from t2 where b>=1 and b<=2000;
select * from t1 join temp_t on (t1.b=temp_t.b);
```

了解了内存表的特性，你就知道了， 其实这里使用内存临时表的效果更好，原因有三个： 

1. 相比于 InnoDB 表，使用内存表不需要写磁盘，往表 temp_t 的写数据的速度更快； 
2.  索引 b 使用 hash 索引，查找的速度比 B-Tree 索引快； 
3.  临时表数据只有 2000 行，占用的内存有限。 

所以只需要把第一条sql 改成

```mysql
create temporary table temp_t(id int primary key, a int, b int, index (b))engine=memory;
```

