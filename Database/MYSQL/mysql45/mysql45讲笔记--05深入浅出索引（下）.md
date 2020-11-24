# 深入浅出索引（下）

## 索引树的搜索和回表

我们先模拟一个场景

```mysql
mysql> create table T (
ID int primary key,
k int NOT NULL DEFAULT 0, 
s varchar(16) NOT NULL DEFAULT '',
index k(k))
engine=InnoDB;

insert into T values(100,1, 'aa'),(200,2,'bb'),(300,3,'cc'),(500,5,'ee'),(600,6,'ff'),(700,7,'gg');
```

下图为InnoDB 的索引组织结构

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-05-01.png)

那我们看一下执行select * from T where k between 3 and 5, 需要执行几次树的搜索，会扫描多少行？

1. 在k索引树上找到k=3的记录，取得id的值是300；
2. 再到ID索引树种查找到D=300对应的值R3；
3. 在k索引树上找到k=5的记录，取得id的值是500;
4. 再到ID索引树种查找到D=500对应的值R5;
5. 在k索引树上找到k=6的记录，不满足条件，循环结束。

在这个过程中**回到主索引树搜索的过程称之为“回表”**。所以一共执行了3+2=5次树搜索，扫描了2+2=4行。

> 为什么需要回表呢？
>
> 答：因为记录的数据是存在主键索引树中的，所以不得不回表。

## 覆盖索引

上面的例子我们了解到 查询n条记录就要回表n次，这样会存在一定的性能消耗，是否可以经过索引优化，避免回表过程呢？

答案是肯定的，我们设想，如果k索引树存储的数据本身就有你要查的字段，这样就不需要回表去查了。如我们执行 select ID from T where k between 3 and 5,这时我们只要查询ID这个字段，而且k索引的叶子就存储了ID的值，这样就可以避免回表过程。也就是说**索引已经覆盖了我们的查询需求，我们称之为覆盖索引**。

**由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引一个常用的性能提升手段**

假设由这样一个场景，有一个高频请求市民身份证查询市民姓名,表结构如下：

```mysql

CREATE TABLE `tuser` (
  `id` int(11) NOT NULL,
  `id_card` varchar(32) DEFAULT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  `ismale` tinyint(1) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `id_card` (`id_card`),
  KEY `name_age` (`name`,`age`)
) ENGINE=InnoDB
```

查询语句`select name from tuser where id_card=300,因为我们有一个id_card搜索引能快速的找到tuser表的id，但是每次都需要回表，再根据id查询name。运用我们覆盖索引的知识，为tuser添加一个联合索引，alert table tuser add index uni_index_name_idcard(id_card,name),这样就可利用覆盖索引，解决回表问题。

## 最左前缀原则

上面利用覆盖原来解决回表问题，你可能会提出，我之前已经有个id_card索引，之后又建立了一个联合索引，这样索引的维护有点高、而且冗余。而且为每一种查询都设计一个索引，这样索引太多了，不易维护而且浪费存储。

其实，**B+树这种索引结构，可以利用索引的“最左前缀”，来定位记录。**

为了更加直观说明最左前缀，我们举个例子：就用（name，age）这个联合索引分析。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-05-02.jpg)

可以看到，索引项是按照索引定义里面出现的字段顺序排序的。

如果当你逻辑需求查到所有的姓名是”张三“的人时，可以快速定位到ID4，然后向后继续查找，直到条件不满足。

如果查询的需求是第一个字”张“，SQL语句的条件是 where name like ‘%张’。这时也能用上这个索引，可以定位到ID3,然后向后继续遍历，直到条件不满足。

可以看到不只是索引的全部定义，只要满足最左前缀，就可以利用索引来加速检索。这个最左前缀可以是联合索引的最左N个字段，也可是字符串索引的最左M个字符。

基于上面的联合索引，我们来讨论一下，在建立联合索引的时候**如何安排索引内的字段顺序。**

- 第一原则是**索引的复用能力**。因为可以支持最左前缀，所有有了（a,b)联合索引后，一般就不需要单独在a建立索引了。因此**第一原则是，如果可以通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的**
- 第二原则是**存储空间**，比如上面的市民表的情况，name字段是比age字段大的，那我就建议你创建一个（name，age）的联合索引和一个（age）的但字段索引，这样可以减少存储空间。

> 所以前面的例子就可以删除id_card 索引了。

## 索引下推

我们还发现又些情况是不属于最左前缀的情况的，我们还是以市民表联合索引（name,age)为例。如果又这样一个需求：查找出”名字第一个字是张，而且年龄是10岁的所有男孩“。

```mysql
mysql> select * from tuser where name like '张%' and age=10 and ismale=1;
```

我们知道能用最左前缀找到第一个满足条件的记录ID3。这样已经过滤掉一部分数据了总比全表扫描好。

然后呢？

当然是判断其他条件是否满足。

在**MySQL5.6之前**，只能从ID3开始一个个回表，到主键索引上找数据，在对比字段值。

而在**MySQL5.6**开始引入了**索引下推优化**（index condition pushdown），可以在索引遍历过程种，**对索引中包含的字段先做判断**，直接过滤掉不满足条件的记录，减少回表次数。

下图是无索引下推流程图：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-05-03.jpg)

下图是有索引下推执行流程图：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/database/mysql/mysql45/mysql45-05-04.jpg)

其中,无索引下推流程图,在（name,age）索引里面特意去掉age的值，这个过程InnoDB并不回去看age的值，只是顺序的把name第一个字是张的记录一条条取出来回表，因此回表4次。

而有索引下推执行流程图,的区别是InnoDB会在（name，age）索引内部就判断age是否等于10，对于不等于10的数据直接判断跳过。所有在例子中ID4，ID5这两条记录需要回表。回表2次。



## 自问自答

1. 在生产中有一种情况是表数据10G而索引确高达30G为什么？怎么解决

   答：这时由于  InnoDB 这种引擎导致的,虽然删除了表的部分记录,但是它的索引还在, 并未释放。解决方法是重建表或者用`alter table T engine=InnoDB`。

