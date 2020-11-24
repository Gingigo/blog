# 45 | 自增id用完怎么办？

MySQL 里有很多自增的 id，每个自增 id 都是定义了初始值，然后不停地往上加步长。虽然自然数没有上限，但是在计算机里面只要定义了表示这个数的字节长度，那它就有上限 

既然自增 id 有上限，就有可能被用完。但是，自增 id 用完了会怎么样呢？

### 表定义自增id

**结果**：

表定义的自增值达到上限后的逻辑是：**再申请下一个 id 时，得到的值保持不变**。

**验证：**

```mysql

create table t(id int unsigned auto_increment primary key) auto_increment=4294967295;
insert into t values(null);
//成功插入一行 4294967295
show create table t;
/* CREATE TABLE `t` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4294967295;
*/

insert into t values(null);
//Duplicate entry '4294967295' for key 'PRIMARY'
```

**分析：**

Int 的长度是4个字节，所以最大可以表示 2^32 - 1（4294967295）不是一个特别大的数 。 对于一个频繁插入删除数据的表来说，是可能会被用完的。

**应对：**

如果有可能超过该上限，id字段需要创建成 8 个字节的 bigint unsigned 

### InnoDB 系统自增 row_id

**结果：**

如果再有插入数据的行为要来申请 row_id，拿到以后再取最后 6 个字节的话就是 0。达到上限后，**下一个值就是 0，然后继续循环**。 

**分析：**

如果你创建的 InnoDB 表没有指定主键，那么 InnoDB 会给你创建一个不可见的，长度为 6 个字节的 row_id 。InnoDB 维护了**一个全局的 dict_sys.row_id 值**，所有无主键的 InnoDB 表，每插入一行数据，都将当前的 dict_sys.row_id 值作为要插入数据的 row_id，然后把 dict_sys.row_id 的值加 1（一个数据库实例公用一个 dict_sys.row_id）

实际上，在代码实现时 row_id 是一个长度为 8 字节的无符号长整型 (bigint unsigned)。但是，InnoDB 在设计时，给 row_id 留的只是 6 个字节的长度，这样写到数据表中时只放了最后 6 个字节，所以 row_id 能写到数据表中的值，就有两个特征： 

-  row_id 写入表中的值范围，是从 0 到 2^48-1；
-  当 dict_sys.row_id=2^48-1时，如果再有插入数据的行为要来申请 row_id，拿到以后再取最后 6 个字节的话就是 0,   然后继续循环。 

**对比**

从这个角度看，我们还是应该在 InnoDB 表中主动创建自增主键。因为，表自增 id 到达上限后，再插入数据时报主键冲突错误，是更能被接受的。（**不报错，还丢失数据**）

###  Xid 

redo log 和 binlog 相配合的时候，提到了它们有一个共同的字段叫作 Xid 。它在 MySQL 中是用来对应事务的。 

**分析**

MySQL 内部维护了一个全局变量 global_query_id，每次执行语句的时候将它赋值给 Query_id，然后给这个变量加 1。如果当前语句是这个事务执行的第一条语句，那么 MySQL 还会同时把 Query_id 赋值给这个事务的 Xid。

**结果**

而 **global_query_id 是一个纯内存变量，重启之后就清零了**。所以你就知道了，在同一个数据库实例中，不同事务的 Xid 也是有可能相同的。

但是 MySQL 重启之后会重新生成新的 binlog 文件，这就保证了，同一个 binlog 文件里，Xid 一定是惟一的。 

**理论上会冲突**

虽然 MySQL 重启不会导致同一个 binlog 里面出现两个相同的 Xid，但是如果 global_query_id 达到上限后，就会继续从 0 开始计数。从理论上讲，还是就会出现同一个 binlog 里面出现相同 Xid 的场景。 

因为 global_query_id 定义的长度是 8 个字节，这个自增值的上限是 2^64-1。要出现这种情况，必须是下面这样的过程： 

1.  执行一个事务，假设 Xid 是 A； 
2.  接下来执行 2^64次查询语句，让 global_query_id 回到 A； 
3.  再启动一个事务，这个事务的 Xid 也是 A。 

 不过，2^64这个值太大了，大到你可以认为这个可能性只会存在于理论上。 

###  Innodb trx_id 

Xid 和 InnoDB 的 trx_id 是两个容易混淆的概念。 **Xid 是由 server 层维护的。InnoDB 内部使用 Xid，就是为了能够在 InnoDB 事务和 server 之间做关联**。但是，InnoDB 自己的 trx_id，是另外维护的。 

> redo log 是 InnoDB的 binlog 是server 层的，就是通过 Xid 关联的。而 InnoDB 独有的事务处理，自然是通过InnoDB 维护的，自然 trx_是由InnoDB另外维护的

**分析**

 InnoDB 内部维护了一个 max_trx_id 全局变量，每次需要申请一个新的 trx_id 时，就获得 max_trx_id 的当前值，然后并将 max_trx_id 加 1。 

> InnoDB 数据可见性的核心思想是：每一行数据都记录了更新它的 trx_id，当一个事务读到一行数据的时候，判断这个数据是否可见的方法，就是通过事务的一致性视图与这行数据的 trx_id 做对比。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-45-01.png)

<center>图1 事务的 trx_id </center>
session B 里，我从 innodb_trx 表里查出的这两个字段，第二个字段 trx_mysql_thread_id 就是线程 id。显示线程 id，是为了说明这两次查询看到的事务对应的线程 id 都是 5，也就是 session A 所在的线程。

可以看到，T2 时刻显示的 trx_id 是一个很大的数；T4 时刻显示的 trx_id 是 1289，看上去是一个比较正常的数字。这是什么原因呢？ 

实际上，在 T1 时刻，session A 还没有涉及到更新，是一个只读事务。而对于只读事务，InnoDB 并不会分配 trx_id。也就是说： 

1.  在 T1 时刻，trx_id 的值其实就是 0。而这个很大的数，只是显示用的。一会儿我会再和你说说这个数据的生成逻辑。 
2. 直到 session A 在 T3 时刻执行 insert 语句的时候，InnoDB 才真正分配了 trx_id。所以，T4 时刻，session B 查到的这个 trx_id 的值就是 1289。

> 需要注意的是，除了显而易见的修改类语句外，如果在 select 语句后面加上 for update，这个事务也是只读事务。

trx_id 有时不只+1：

1. update 和 delete 语句除了事务本身，还涉及到标记删除旧数据，也就是要把数据放到 purge 队列里等待后续物理删除，这个操作也会把 max_trx_id+1， 因此在一个事务中至少加 2； 
2. InnoDB 的后台操作，比如表的索引信息统计这类操作，也是会启动内部事务的，因此你可能看到，trx_id 值并不是按照加 1 递增的。

> T2 时刻查到的这个很大的数字是怎么来的呢？
>
> 算法是：把当前事务的 trx 变量的指针地址转成整数，再加上 2^48。使用这个算法，就可以保证以下两点： 
>
> 1.  因为同一个只读事务在执行期间，它的指针地址是不会变的，所以不论是在 innodb_trx 还是在 innodb_locks 表里，同一个只读事务查出来的 trx_id 就会是一样的。 
> 2.  如果有并行的多个只读事务，每个事务的 trx 变量的指针地址肯定不同。这样，不同的并发只读事务，查出来的 trx_id 就是不同的。 
> 3. 在显示值里面加上 2^48，目的是要保证只读事务显示的 trx_id 值比较大，正常情况下就会区别于读写事务的 id。

**只读事务不分配 trx_id，有什么好处呢？** 

- 一个好处是，**这样做可以减小事务视图里面活跃事务数组的大小**。因为当前正在运行的只读事务，是不影响数据的可见性判断的。所以，在创建事务的一致性视图时，InnoDB 就只需要拷贝读写事务的 trx_id。
-  另一个好处是，**可以减少 trx_id 的申请次数**。在 InnoDB 里，即使你只是执行一个普通的 select 语句，在执行过程中，也是要对应一个只读事务的。所以只读事务优化后，**普通的查询语句不需要申请 trx_id，就大大减少了并发事务申请 trx_id 的锁冲突**。 

**结论**

max_trx_id 会持久化存储，重启也不会重置为 0，那么从理论上讲，只要一个 MySQL 服务跑得足够久，就可能出现 max_trx_id 达到 2^48-1 的上限，然后从 0 开始的情况，重新开始循环。

当达到这个状态后，MySQL 就会持续出现一个脏读的 bug，我们来复现一下这个 bug。 

**这个 bug 也是只存在于理论上？** ，假设一个 MySQL 实例的 TPS 是每秒 50 万，持续这个压力的话，在 17.8 年后，就会出现这个情况,不过，这个 bug 是只要 MySQL 实例服务时间够长，就会必然出现的。 

###  thread_id 

thread_id 的逻辑很好理解：系统保存了一个全局变量 thread_id_counter，每新建一个连接，就将 thread_id_counter 赋值给这个新连接的线程变量。

**分析**

thread_id_counter 定义的大小是 4 个字节，因此达到 2^32-1 后，它就会重置为 0，然后继续增加。但是，你不会在 show processlist 里看到两个相同的 thread_id。

thread_id_counter 定义的大小是 4 个字节，因此达到 232-1 后，它就会重置为 0，然后继续增加。但是，你不会在 show processlist 里看到两个相同的 thread_id。

 这，是因为 MySQL 设计了一个**唯一数组的逻辑**，给新线程分配 thread_id 的时候，逻辑代码是这样的： 

```mysql
do {
  new_id= thread_id_counter++;
} while (!thread_ids.insert_unique(new_id).second);
```

### 小结

每种自增 id 有各自的应用场景，在达到上限后的表现也不同： 

1.  表的自增 id 达到上限后，再申请时它的值就不会改变，进而导致继续插入数据时报主键冲突的错误。 （推荐）
2.  row_id 达到上限后，则会归 0 再重新递增，如果出现相同的 row_id，后写的数据会覆盖之前的数据。 （不推荐 覆盖数据还不报错）
3.  Xid 只需要不在同一个 binlog 文件中出现重复值即可。虽然理论上会出现重复值，但是概率极小，可以忽略不计。 
4.  InnoDB 的 max_trx_id 递增值每次 MySQL 重启都会被保存起来，所以我们文章中提到的脏读的例子就是一个必现的 bug，好在留给我们的时间还很充裕。 
5.  thread_id 是我们使用中最常见的，而且也是处理得最好的一个自增 id 逻辑了。 
