# 39 | 自增主键为什么不是连续的？

### 自增值保存在哪里？

为了方便量化分析，举个例子

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB;
```

通过执行 `show create  t`，能查看到自增值得大小，你或许会说 InnoDB 引擎得表的自增值保存在表结构中。实际上，**表的结构定义存放在后缀名为 .frm 的文件中，但是并不会保存自增值。**

不同的引擎对于自增值的保存策略不同：

- MyISAM 引擎的自增值保存在数据文件中。
- InnoDB 引擎的自增值，其实是保存在内存里
  - MySQL 8.0 后才有了“自增值持久化”的能力。通过将自增值变更记录在 redo log 中，重启的时候依靠 redo log 恢复重启之前的值。
  - MySQL 5.7 及之前的版本，自增值保存在内存里，并没有持久化。每次重启后，第一次打开表的时候，都会去找自增值得最大值max（id），然后max（id）+ 1 作为当前表的自增值。

### 自增值修改机制

在 MySQL 中，如果字段 id 被定义为 AUTO_INCREMENT,在插入一行的时候，自增值得行为如下：

1. 如果插入数据时 id 字段指定为 0 、null 或者为指定，那么就把这个表当前的 AUTO_INCREMENT 值填到自增字段；
2. 如果插入数据时 id 字段指定了具体的值，就直接使用语句里指定的值。(假设当前插入值是 X，当前的自增值是 Y。那个大，选择哪一个)
   - 如果 X < Y,那么这个表的自增值不变，即 选Y；
   - 如果 X > Y,就需要把当前自增值修改为新的自增值，即 选 X 且 修改 Y 为 X

**两个参数**

<font color='orange'>auto_increment_offset</font>:   表示自增初始值，默认为 0；

<font color='orange'> auto_increment_increment </font>：表示自增步长，默认为 0；

> 备注：在一些场景下，使用的就不全是默认值。比如，双 M 的主备结构里要求双写的时候，我们就可能会设置成 auto_increment_increment=2，让一个库的自增 id 都是奇数，另一个库的自增 id 都是偶数，避免两个库生成的主键发生冲突。

### 自增值的修改时机

假设以下的 auto_increment_offset 和 auto_increment_increment 都是使用默认值。

#### 不连续原因

**情况一：唯一键冲突是导致自增主键 id 不连续的第一种原因。**

假设，表 t 里面已经有了（1，1，1）这条记录，这时在执行下面这条语句：

```mysql
insert into t values(null,1,1);
```

执行流程如下：

1. 执行器调用 InnoDB 引擎接口写入一行，传入的这一行的值是 (0,1,1);
2. InnoDB 发现用户没有指定自增 id 的值，获取表 t 当前的自增值 2； 
3. 将传入的行的值改成 (2,1,1);
4.  将表的自增值改成 3； 
5. 继续执行插入数据操作，由于已经存在 c=1 的记录，所以报 Duplicate key error，语句返回。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-39-01.png)

<center> 图1 insert(null, 1,1) 唯一键冲突 </center>

可以看到，这个表的自增值改成 3，是在真正执行插入数据的操作之前。这个语句真正执行的时候，因为碰到唯一键 c 冲突，所以 id=2 这一行并没有插入成功，但也没有将自增值再改回去。

所以，在这之后，再插入新的数据行时，拿到的自增 id 就是 3。也就是说，出现了自增主键不连续的情况。

 **情况二：事务回滚也会产生类似的现象，这就是第二种原因**

```mysql
insert into t values(null,1,1);
begin;
insert into t values(null,2,2);
rollback;
insert into t values(null,2,2);
//插入的行是(3,2,2)
```

**情况三：批量插入语句有可能会导致主键 id 出现自增 id 不连续的第三种原因 **

预先不知道要申请多少个自增 id，那么一种直接的想法就是需要一个时申请一个。但如果一个 select … insert 语句要插入 10 万行数据，按照这个逻辑的话就要申请 10 万次。显然，这种申请自增 id 的策略，在大批量插入数据的情况下，不但速度慢，还会影响并发插入的性能。

 因此，对于批量插入数据的语句，MySQL 有一个批量申请自增 id 的策略： 

1.  语句执行过程中，第一次申请自增 id，会分配 1 个； 
2. 1 个用完以后，这个语句第二次申请自增 id，会分配 2 个； 
3. 2 个用完以后，还是这个语句，第三次申请自增 id，会分配 4 个； 
4.  依此类推，同一个语句去申请自增 id，每次申请到的自增 id 个数都是上一次的两倍。 

 举个例子，我们一起看看下面的这个语句序列： 

```msyql

insert into t values(null, 1,1);
insert into t values(null, 2,2);
insert into t values(null, 3,3);
insert into t values(null, 4,4);
create table t2 like t;
insert into t2(c,d) select c,d from t;
insert into t2 values(null, 5,5);
```

insert…select，实际上往表 t2 中插入了 4 行数据。但是，这四行数据是分三次申请的自增 id，第一次申请到了 id=1，第二次被分配了 id=2 和 id=3， 第三次被分配到 id=4 到 id=7。

由于这条语句实际只用上了 4 个 id，所以 id=5 到 id=7 就被浪费掉了。之后，再执行 insert into t2 values(null, 5,5)，实际上插入的数据就是（8,5,5)。

#### 自增值为什么不能退回

假设有两个并行执行的事务，在申请自增值的时候，为了避免两个事务申请到相同的自增 id，肯定要加锁，然后顺序申请。 

1. 假设事务 A 申请到了 id=2， 事务 B 申请到 id=3，那么这时候表 t 的自增值是 4，之后继续执行。 
2. 事务 B 正确提交了，但事务 A 出现了唯一键冲突。
3.  如果允许事务 A 把自增 id 回退，也就是把表 t 的当前自增值改回 2，那么就会出现这样的情况：表里面已经有 id=3 的行，而当前的自增 id 值是 2。 
4.  接下来，继续执行的其他事务就会申请到 id=2，然后再申请到 id=3。这时，就会出现插入语句报错“主键冲突”。 

**解决这个主键冲突，有两种方法：** 

1. 每次申请 id 之前，先判断表里面是否已经存在这个 id。如果存在，就跳过这个 id。但是，这个方法的成本很高。因为，本来申请 id 是一个很快的操作，现在还要再去主键索引树上判断 id 是否存在。 
2.  把自增 id 的锁范围扩大，必须等到一个事务执行完成并提交，下一个事务才能再申请自增 id。这个方法的问题，就是锁的粒度太大，系统并发能力大大下降。 

可见，这两个方法都会导致性能问题。造成这些麻烦的罪魁祸首，就是我们假设的这个“允许自增 id 回退”的前提导致的。 

因此，InnoDB 放弃了这个设计，语句执行失败也不回退自增 id。也正是因为这样，所以才只保证了自增 id 是递增的，但不保证是连续的。 

###  自增锁的优化 

- MySQL 5.0 及之前 ，自增锁的范围是语句级别。也就是说，如果一个语句申请了一个表自增锁，这个锁会等语句执行结束以后才释放。显然，这样设计会影响并发度。
- MySQL 5.1.22 版本引入了一个新策略，新增参数 innodb_autoinc_lock_mode，默认值是 1。
  1. 这个参数的值被设置为 0 时，表示采用之前 MySQL 5.0 版本的策略，即语句执行结束后才释放锁； 
  2. 这个参数的值被设置为 1 时： 
     - 普通 insert 语句，自增锁在申请之后就马上释放；
     - 类似 insert … select 这样的批量插入数据的语句，自增锁还是要等语句结束后才被释放； 
  3. 这个参数的值被设置为 2 时，所有的申请自增主键的动作都是申请后就释放锁。 

**为什么默认设置下，insert … select 要使用语句级的锁？ **

答案是，这么设计还是为了数据的一致性。

 我们一起来看一下这个场景： 

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-39-02.png)

<center>图2  批量插入数据的自增锁 </center>

设想一下，如果 session B 是申请了自增值以后马上就释放自增锁，那么就可能出现这样的情况：

-  session B 先插入了两个记录，(1,1,1)、(2,2,2)； 
-  然后，session A 来申请自增 id 得到 id=3，插入了（3,5,5)； 
-  之后，session B 继续执行，插入两条记录 (4,3,3)、 (5,4,4)。 

从数据逻辑上看是对的。但是，如果我们现在的 binlog_format=statement，你可以设想下，binlog 会怎么记录呢？

由于两个 session 是同时执行插入数据命令的，所以 binlog 里面对表 t2 的更新日志只有两种情况：要么先记 session A 的，要么先记 session B 的。

假设 binlog_format=statement 。但不论是哪一种（先记录session A 还是先记录 session B），这个 binlog 拿去从库执行，或者用来恢复临时实例，备库和临时实例里面，session B 这个语句执行出来，生成的结果里面，id 都是连续的。这时，这个库就发生了数据不一致。

其实，这是因为原库 session B 的 insert 语句，生成的 id 不连续。这个不连续的 id，用 statement 格式的 binlog 来串行执行，是执行不出来的。

 而要解决这个问题，有两种思路： 

1.  一种思路是，让原库的批量插入数据语句，固定生成连续的 id 值。所以，自增锁直到语句执行结束才释放，就是为了达到这个目的。 
2. 另一种思路是，在 binlog 里面把插入数据的操作都如实记录进来，到备库执行的时候，不再依赖于自增主键去生成。这种情况，其实就是 innodb_autoinc_lock_mode 设置为 2，同时 binlog_format 设置为 row。

**因此，在生产上，尤其是有 insert … select 这种批量插入数据的场景时，从并发插入数据性能的角度考虑，我建议你这样设置：innodb_autoinc_lock_mode=2 ，并且 binlog_format=row. 这样做，既能提升并发性，又不会出现数据一致性问题。**

 需要注意的是，**这里说的批量插入数据，包含的语句类型是 insert … select、replace … select 和 load data 语句。 **
