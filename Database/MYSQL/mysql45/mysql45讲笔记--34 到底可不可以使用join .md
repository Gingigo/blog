# 34 | 到底可不可以使用join？

 在实际生产中，关于 join 语句使用的问题，一般会集中在以下两类： 

1.  我们 DBA 不让使用 join，使用 join 有什么问题呢？ 
2.  如果有两个大小不同的表做 join，应该用哪个表做驱动表呢？ 

量化分析，举个例子：

```mysql
CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<=100)
```

### Index Nested-Loop Join （NLJ）

```mysql
select * from t1 straight_join t2 on (t1.a=t2.a);
```

> 注意：
>
> 如果直接使用 join 语句，MySQL 优化器可能会选择表 t1 或 t2 作为驱动表，这样会影响我们分析 SQL 语句的执行过程
>
> straight_join 是使用固定方向连接方式执行查询，这样第一个表就是驱动表，第二个表是被驱动表

我们来看一下这条语句的 explain 结果：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-34-01.png)

<center> 图 1 使用索引字段 join 的 explain 结果 </center>
这条语句的**执行流程**：

1. 从表 t1 中读入一行数据 R；
2. 从数据行中，取出 a 字段到表 t2 里查找；
3. 取出表 t2 中满足条件的行，跟 R 组成一行，作为结果集的一部分；
4. 重复执行步骤 1 到 3，直到表 t1 的末尾循环结束。

这个过程是先遍历表 t1，然后根据从表 t1 中取出的每行数据中的 a 值，去表 t2 中查找满足条件的记录。在形式上，这个过程就跟我们写程序时的嵌套查询类似，并且可以用上被驱动表的索引，**所以我们称之为“Index Nested-Loop Join”，简称 NLJ**。

###  单表查询 

假设不使用 join，那我们就只能用单表查询。我们看看上面这条语句的需求，用单表查询怎么实现。

1. 执行 select *  from t1,查出表 t1 的所有数据，这里有 100 行；
2. 循环遍历这 100 行数据：
   - 从每一行 R 取出字段 a 的值 $R.a;
   - 执行 select * from t2 where a=$R.a;
   - 把返回的结果和 R 构成结果集的一行。

可以看到，在这个查询过程，也是扫描了 200 行，但是总共执行了 101 条语句，比直接 join 多了 100 次交互。除此之外，客户端还要自己拼接 SQL 语句和结果。

显然，这么做还不如直接 join 好。

**复杂度计算**

在这个 join 语句执行过程中，驱动表是走全表扫描，而被驱动表是走树搜索。

假设被驱动表的行数是 M。每次在被驱动表查一行数据，要先搜索索引 a，再搜索主键索引。每次搜索一棵树近似复杂度是以 2 为底的 M 的对数，记为 log2M，所以在被驱动表上查一行的时间复杂度是 2*log2M。

假设驱动表的行数是 N，执行过程就要扫描驱动表 N 行，然后对于每一行，到被驱动表上匹配一次。

因此整个执行过程，近似复杂度是 N + N*2*log2M。

**显然，N 对扫描行数的影响更大，因此应该让小表来做驱动表。**

**小结**

1. 使用 join 语句，性能比强行拆成多个单表执行 SQL 语句的性能要好； 
2.  如果使用 join 语句的话，需要让小表做驱动表。 

**但是，这里的前提是“可以使用被驱动表的索引”**

### Simple Nested-Loop Join （NLJ）

 接下来，我们再看看被驱动表用不上索引的情况。 

 现在，我们把 SQL 语句改成这样： 

```mysql
select * from t1 straight_join t2 on (t1.a=t2.b);
```

由于表 t2 的字段 b 上没有索引,每次到 t2 去匹配的时候需要做一次全表扫描。

如果只看结果的话，这个算法是正确的，而且这个算法也有一个名字，**叫做“Simple Nested-Loop Join”**。

**复杂度计算**

但是，这样算来，这个 SQL 请求就要扫描表 t2 多达 100 次，总共扫描 100*1000=10 万行。

很明显，这个算法太过笨重了，MySQL 也没有使用这个 Simple Nested-Loop Join 算法，而是使用了另一个叫作“Block Nested-Loop Join”的算法，简称 BNL。

### Block Nested-Loop Join （BNJ）

 这时候，被驱动表上没有可用的索引，算法的流程是这样的： 

1. 把表 t1 的数据读入线程内存 join_buffer 中，由于我们这个语句中写的是 select *因此是把整个表 t1 放入了内存；
2. 扫描表 t2，把表 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回。 

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-34-02.png)

<center>图 2 Block Nested-Loop Join 算法的执行流程</center>
 对应地，这条 SQL 语句的 explain 结果如下所示： 

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-34-03.png)

<center> 图 3 不使用索引字段 join 的 explain 结果 </center>
可以看到，在这个过程中，对表 t1 和 t2 都做了一次全表扫描，因此总的扫描行数是 1100。由于 join_buffer 是以无序数组的方式组织的，因此对表 t2 中的每一行，都要做 100 次判断，总共需要在内存中做的判断次数是：100*1000=10 万次。

前面我们说过，如果使用 Simple Nested-Loop Join 算法进行查询，扫描行数也是 10 万行。因此，从时间复杂度上来说，这两个算法是一样的。但是，Block Nested-Loop Join 算法的这 10 万次判断是内存操作，速度上会快很多，性能也更好。

**复杂度计算** 

假设小表的行数是 N，大表的行数是 M，那么在这个算法里： 

1.  两个表都做一次全表扫描，所以总的扫描行数是 M+N； 
2. 内存中的判断次数是 M*N。

所以无论是按个表作为驱动表，时间复杂度都是一样的。前提是 join_buffer 足够大。

#### join_buffer 的影响

join_buffer 的大小是由参数 join_buffer_size 设定的，默认值是 256k。**如果放不下表 t1 的所有数据话**，策略很简单，就是分段放。我把 join_buffer_size 改成 1200，再执行。

```msyql
select * from t1 straight_join t2 on (t1.a=t2.b);
```

执行过程就变成了： 

1. 扫描表 t1，顺序读取数据行放入 join_buffer 中，放完第 88 行 join_buffer 满了，继续第 2 步；
2.  扫描表 t2，把 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回； 
3.  清空 join_buffer； 
4. 继续扫描表 t1，顺序读取最后的 12 行数据放入 join_buffer 中，继续执行第 2 步。

这个流程才体现出了这个算法名字中“Block”的由来，表示“分块去 join”。

**复杂度计算**

可以看到，这时候由于表 t1 被分成了两次放入 join_buffer 中，导致表 t2 会被扫描两次。虽然分成两次放入 join_buffer，但是判断等值条件的次数还是不变的，依然是 (88+12)*1000=10 万次。

我们再来看下，在这种情况下驱动表的选择问题。

假设，驱动表的数据行数是 N，需要分 K 段才能完成算法流程，被驱动表的数据行数是 M。 

注意，这里的 K 不是常数，N 越大 K 就会越大，因此把 K 表示为λ*N，显然λ的取值范围是 (0,1)。

所以，在这个算法的执行过程中： 

1. 扫描行数是 N+λ*N*M； 
2. 内存判断 N*M 次。 

显然，内存判断次数是不受选择哪个表作为驱动表影响的。而考虑到扫描行数，在 M 和 N 大小确定的情况下，N 小一些，整个算式的结果会更小。

**所以结论是，应该让小表当驱动表。** 

理解了 MySQL 执行 join 的两种算法，现在我们再来试着回答文章开头的两个问题。

第一个问题：能不能使用 join 语句？ 

1. 如果可以使用 Index Nested-Loop Join 算法，也就是说可以用上被驱动表上的索引，其实是没问题的； 
2. 如果使用 Block Nested-Loop Join 算法，扫描行数就会过多。尤其是在大表上的 join 操作，这样可能要扫描被驱动表很多次，会占用大量的系统资源。所以这种 join 尽量不要用。

所以你在判断要不要使用 join 语句时，就是看 explain 结果里面，Extra 字段里面有没有出现“Block Nested Loop”字样。

 第二个问题是：如果要使用 join，应该选择大表做驱动表还是选择小表做驱动表？ 

1.  如果是 Index Nested-Loop Join 算法，应该选择小表做驱动表； 
2.  如果是 Block Nested-Loop Join 算法： 
   - 在 join_buffer_size 足够大的时候，是一样的；
   -  在 join_buffer_size 不够大的时候（这种情况更常见），应该选择小表做驱动表。 

### 什么叫作“小表”

我们前面的例子是没有加条件的。如果我在语句的 where 条件加上 t2.id<=50 这个限定条件，再来看下这两条语句： 

```mysql
select * from t1 straight_join t2 on (t1.b=t2.b) where t2.id<=50;
select * from t2 straight_join t1 on (t1.b=t2.b) where t2.id<=50;
```

注意，为了让两条语句的被驱动表都用不上索引，所以 join 字段都使用了没有索引的字段 b。

但如果是用第二个语句的话，join_buffer 只需要放入 t2 的前 50 行，显然是更好的。所以这里，“t2 的前 50 行”是那个相对小的表，也就是“小表”。

 我们再来看另外一组例子： 

```mysql
select t1.b,t2.* from  t1  straight_join t2 on (t1.b=t2.b) where t2.id<=100;
select t1.b,t2.* from  t2  straight_join t1 on (t1.b=t2.b) where t2.id<=100;
```

 这个例子里，表 t1 和 t2 都是只有 100 行参加 join。但是，这两条语句每次查询放入 join_buffer 中的数据是不一样的： 

-  表 t1 只查字段 b，因此如果把 t1 放到 join_buffer 中，则 join_buffer 中只需要放入 b 的值； 
-  表 t2 需要查所有的字段，因此如果把表 t2 放到 join_buffer 中的话，就需要放入三个字段 id、a 和 b。 

这里，我们应该选择表 t1 作为驱动表。也就是说在这个例子里，“只需要一列参与 join 的表 t1”是那个相对小的表。

 所以，更准确地说，**在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表**。 
