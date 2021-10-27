## mysql高级优化

### 问题1：存储引擎介绍

查看命令：show engines;

| 对比项   | MyISAM                                                   | InnoDB                                                       |
| -------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| 主外键   | 不支持主外键                                             | 支持主外键                                                   |
| 事务     | 不支持事务                                               | 支持事务                                                     |
| 行表锁   | 表锁，即使操作一条记录也会锁住整个表，不适合高并发的操作 | 行锁，操作时只锁一行，不对其他行有影响，适合高并发的操作     |
| 缓存     | 只缓存索引，不缓存真实数据                               | 不仅缓存索引还要缓存真实数据，对内存要求较高，而且内存大小对性能有决定性的影响 |
| 表空间   | 小                                                       | 大                                                           |
| 关注点   | 性能                                                     | 事务                                                         |
| 默认安装 | 是                                                       | 是                                                           |

### 问题2：SQL性能下降的原因

SQL执行时间长，等待时间长的原因：

查询语句写的烂；索引失效；关联查询太多join（设计缺陷或者不合理的需求），服务器调优以及各个参数设置（缓存，线程数等）

### 问题3：SQL执行加载顺序

```
select distinct ...                  5

from ...  join ... on ...            1 

where ...                            2
 
group by ...                         3

having ...                           4

order by ...                         6

limit ...                            7
```



### 问题4：七种join的编写

```
第一种：A与B的交集

select e.*,d.* from emp e
inner join dept d on e.deptno = d.deptno;

第二种：A的全部

select e.*,d.* from emp e
left join dept d on e.deptno = d.deptno;

第三种：B的全部

select e.*,d.* from emp e
right join dept d on e.deptno = d.deptno;

第四种：包含A，但不包含B

select e.*,d.* from emp e
left join dept d on e.deptno = d.deptno where d.deptno is null;

第五种：包含B，但不包含A

select e.*,d.* from emp e
right join dept d on e.deptno = d.deptno where e.empno is null;

第六种：A与B的并集

select e.*,d.* from emp e
left join dept d on e.deptno = d.deptno
union
select e.*,d.* from emp e
right join dept d on e.deptno = d.deptno

第七种：A与B的并集-A与B的交集

select e.*,d.* from emp e
left join dept d on e.deptno = d.deptno where d.deptno is null
union 
select e.*,d.* from emp e
right join dept d on e.deptno = d.deptno where e.empno is null;
```



### 问题5：索引是什么

MySQL官方对索引的定义为：**索引（index）是帮助MySQL高效获取数据的数据结构。**

可以得到索引的本质：**索引是一种数据结构。可以简单理解为：排好序的快速查找数据结构。**

在数据之外，数据库系统还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据，这样就可以在这些数据结构上实现高级查找算法。这种数据结构，就是索引。下图就是一种可能的索引方式示例：
![image-20211016194323728](C:\Users\陈泽杰\AppData\Roaming\Typora\typora-user-images\image-20211016194323728.png)

左边是数据表，一共有两列七条记录，最左边的是数据记录的物理地址。

为了加快Col2的查找，可以维护一个右边所示的二叉查找树，每个节点分别包含索引键值和一个指向对应数据记录物理地址的指针，这样就可以运用二叉查找在一定的复杂度内获取到相应数据，从而快速地检索出符合条件的记录。

结论：**数据本身之外，数据库还维护着一个满足特定查找算法的数据结构，这些数据以某种方式指向数据，这样就可以在这些数据结构的基础上实现高级查找算法，这种数据结构就是索引。（BTree，即B树索引）**

一般来说索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储在磁盘上。

我们平常所说的索引，如果没有特别说明，都指的是B树（多路搜索树，不一定是二叉树）结构组织的索引。其中，聚集索引，次要索引，覆盖索引，复合索引，前缀索引，唯一索引默认都是使用B+树索引，统称索引。当然，除了B+树这种类型的索引之外，还有哈希索引（hash index）等。

### 问题6：索引的优势和劣势

索引优势：类似大学图书馆建书目索引，提高数据库的检索效率，降低数据库的IO成本；通过索引列对数据进行排序，降低数据排序的成本，降低CPU的消耗。

索引劣势：实际上索引也是一张表，该表保存了主键和索引字段，并指向实体表的记录，所以索引列也是需要占用空间的。虽然索引大大提高了查询速度，但同时会降低更新表的速度，例如对表进行insert、update和delete。因为更新表时，MySQL不仅要保存真实数据，还要保存一下索引文件每次更新添加了索引列的字段，都会调整因为更新带来的键值变化后的索引信息。

索引只是提高效率的一个因素，如果你的MySQL有大数据量的表，就需要花时间建立最优秀的索引，或者优化查询。

### 问题7：MySQL索引的分类

单值索引：即一个索引只包含单个列，一个表可以有多个单值索引

唯一索引：索引列的值必须唯一，但允许有空值

复合索引：即一个索引包含多个列

基本语法：

```
创建：CREATE [UNIQUE] INDEX indexName ON mytable(columnname(length));

ALTER mytable ADD [UNIQUE] INDEX [indexname] ON (columnname(length));

删除：DROP INDEX [indexName] ON mytable;

查看：SHOW INDEX FROM table_name

有四种方式来添加数据表的索引：
ALTER TABLE tbl_name ADD PRIMARY KEY(column_list)：该语句添加一个主键，这意味着索引值必须是唯一的，且不能为null。

ALTER TABLE tbl_name ADD UNIQUE index_name(column_list)：这条语句创建的索引值必须是唯一的（除了null之外，null可能会出现多次）。

ALTER TABLE tbl_name ADD INDEX index_name(column_list)：添加普通索引，索引值可出现多次。

ALTER TABLE tbl_name ADD FULLTEXT index_name(column_list)：该语句指定了索引为FULLTEXT，用于全文索引。
```

### 问题7：MySQL索引结构

BTree索引、Hash索引、full-text全文索引、R-Tree索引

### 问题8：哪些情况需要创建索引

1. 主键自动建立唯一索引
2. 查询中与其他表关联的字段，外键关系建立索引（员工表和部门表的deptno）
3. 频繁作为查询条件的字段应该创建索引（比如：银行系统的银行账号，电信系统的手机号，微信系统的微信号）
4. 查询中经常排序的字段，排序字段若通过索引去访问将大大提高排序速度
5. 查询中统计或者分组字段
6. 单键/组合索引的选择问题，who？单键索引。（在高并发的条件下倾向于创建组合索引）

### 问题9：哪些情况不需要创建索引

1. 表记录太少，不需要创建索引（一般超过300万条记录就会建立索引）
2. 频繁更新的字段不适合创建索引（因为每次更新不单单是更新了记录还会更新索引文件）
3. 经常增删改的表（提高了查询速度，同时却降低了更新表的速度，因为更新表时，mysql不仅要保存数据，还要保存一下索引文件）
4. where条件里用不到的字段不要创建索引
5. 如果某个列的包含许多重复的内容，即数据重复且分布均匀的字段，就没必要为它建立索引。（例如：国籍字段：中国，性别字段：男/女），索引的选择性是指索引列中不同值的数目和表中的记录数的比值，如果一个表中有2000条记录，表索引列有1980个不同的值，那么这个索引的选择性就是1980/2000=0.99。一个索引的选择性越接近于1，这个索引的效率就越高。

### 问题10：MySQL的常见瓶颈

1. CPU：CPU在饱和的时候一般发生在数据装入内存或从磁盘上读取数据的时候
2. IO：磁盘I/O瓶颈发生在装入数据远大于内存容量的时候
3. 服务器硬件（调优配置类的设置等）的性能瓶颈：top，free，iostat和vmstat来查看系统的性能状态

### 问题11：Explain是什么

explain：查看执行计划，使用explain关键字可以模拟MySQL内部的查询优化器执行SQL查询语句，从而知道MySQL是如何处理你的SQL语句的。explain用来分析我们写的查询语句或者是表结构是否存在性能瓶颈。

### 问题12：Explain怎么用

explain+SQL语句：查看执行计划包含的信息

| id   | select_type | table | type | possible_keys | key  | key_len | ref  | lows | Extra |
| ---- | ----------- | ----- | ---- | ------------- | ---- | ------- | ---- | ---- | ----- |
|      |             |       |      |               |      |         |      |      |       |

#### id：select查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序

三种情况：

1. id相同，执行顺序由上至下

```
explain select t2.*
from t1,t2,t3
where t1.id = t2.id and t1.id = t3.id
and t1.other_colunm = '';
```

运行结果：id相同的情况下，加载表的顺序由上至下，也就是t1、t3、t2，先加载t1表，接着加载t3表，最后加载t2表。

```
id    select_type  table

1	  SIMPLE	    t1
1     SIMPLE        t3
1     SIMPLE        t2
```

2. id不同，如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行

```
explain select t2.*
from t2
where id = (select id 
              from t1
              where id = (select t3.id 
              	            from t3 
              	            where t3.other_column = ''));
```

运行结果：如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行。

表的读取顺序为t3、t1、t2

```
id     select_type   table

1      PRIMARY       t2
2      SUBQUERY      t1
3      SUBQUERY      t3
```

3. id相同不同，同时存在

```
explain select t2.* from (
select t3.id
from t3 
where t3.other_column = '') s1, t2
where s1.id = t2.id;
```

运行结果：id如果相同，可以认为是一组，从上往下顺序执行；在所有组中，id值越大，优先级越高，越先执行。

表的读取顺序为：t3、<derived2>（由id为2的select子句衍生出来的表，即s1）、t2

```
id     select_type    table

1      PRIMARY        <derived2>
1      PRIMARY        t2
2      DERIVED        t3
```

#### select_type：查询的类型

1. select_type有哪些类型？

```
id     select_type
1      SIMPLE
2      PRIMARY
3      SUBQUERY
4      DERIVED
6      UNION
6      UNION RESULT
```

2. select_type表示查询的类型，主要是用于区别普通查询、联合查询、子查询等复杂查询

| SIMPLE       | 简单的select查询，查询中不包含子查询或者UNION                |
| ------------ | ------------------------------------------------------------ |
| PRIMARY      | 查询中若包含任何复杂的子部分，最外层查询则被标记为PRIMARY    |
| SUBQUERY     | 在select或where列表中包含了子查询                            |
| DERIVED      | 在from列表中包含的子查询则被标记为DERIVED（衍生），MySQL会递归执行这些子查询，把结果放在临时表里 |
| UNION        | 若第二个select出现在union之后，则被标记为union；若union包含在from子句的子查询中，外层select将被标记为：DERIVED |
| UNION RESULT | 从UNION表获取结果的select                                    |

#### table：显示这一行的数据是关于哪张表的

#### type：访问类型排列

| ALL  | index | range | ref  | eq_ref | const | system | NULL |
| ---- | ----- | ----- | ---- | ------ | ----- | ------ | ---- |
|      |       |       |      |        |       |        |      |

type显示的是访问类型，是较为重要的一个指标，显示查询使用了哪种类型，结果值从最好到最差依次是：

system > const > eq_ref > ref > range > index > ALL

system：表只有一行记录（等于系统表），这是const类型的特例，平时不会出现，这个也可以忽略不计。

const：表示通过索引一次就找到了，const用于比较primary key或者unique索引。因为只匹配一行数据，所以很快。如将主键置于where列表中，MySQL就能将该查询转换为一个常量。

eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键或唯一索引扫描。

ref：非唯一性索引扫描，返回匹配某个单独值的所有行。本质上也是一种索引访问，它返回所有匹配

const：表示通过索引一次就找到了，const用于比较primary key或者unique索引。因为只匹配一行数据，所以很快。如将主键置于where列表中，MySQL就能将该查询转换为一个常量。

eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键或唯一索引扫描。

ref：非唯一性索引扫描，返回匹配某个单独值的所有行。本质上也是一种索引访问，它返回所有匹配某个单独值的行，然而，它可能会找到多个符合条件的行，所以他应该属于查找和扫描的混合体。

range：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般就是在你的where语句中出现了between、<、>、in等的查询。这种范围扫描索引扫描比全表扫描要好，因为它只需要开始于索引的某一点，而结束于另一点，不用扫描全部索引。

index：Full Index Scan，index与All区别为index类型只遍历索引树。这通常比ALL快，因为索引文件通常比数据文件小。也就是说虽然all和index都是读全表，但index是从索引中读取的，而all是从硬盘中读的。

all：Full Table Scan，将遍历全表以找到匹配的行。

一般来说，得保证查询至少达到range级别，最好能达到ref。

#### possible_keys：

显示可能应用在这张表中的索引，一个或多个。查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询实际使用。

#### keys：

实际使用的索引。如果为NULL，则没有使用索引。查询中若使用了覆盖索引，则该索引仅出现在key列表中

#### key_len：

表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。在不损失精确性的情况下，长度越短越好。key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的。

#### ref：

显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值。

#### rows：

根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数。

#### Extra：



### **问题13：Explain能干嘛**

1. 表的读取顺序：id
2. 数据读取操作的操作类型：select_type
3. 哪些索引可以使用
4. 哪些索引被实际使用
5. 表之间的引用
6. 每张表有多少行被优化器查询

### **问题14：**