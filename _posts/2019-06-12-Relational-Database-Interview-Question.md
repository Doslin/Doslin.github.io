---
layout: post
title: '关系型数据库面试题'
description: "关系型数据库面试题"
author: qiuzhilin
tags: 
  - OS
last_modified_at: 2019-06-12T13:46:19-05:00
---

## 1. 存储引擎

### 1.1. mysql 有哪些存储引擎？有什么区别？

- **InnoDB** - Mysql 的默认事务型存储引擎。性能不错且支持自动崩溃恢复。
- **MyISAM** - Mysql 5.1 版本前的默认存储引擎。特性丰富但不支持事务，也没有崩溃恢复功能。
- **CSV** - 可以将 CSV 文件作为 Mysql 的表来处理，但这种表不支持索引。
- **Memory** - 适合快速访问数据，且数据不会被修改，重启丢失也没有关系。
- **NDB** - 用于 Mysql 集群场景。

## 2. 索引

### 2.1. 数据库索引有哪些数据结构？

- B-Tree
- B+Tree
- Hash

#### 2.1.1. B-Tree

一棵 M 阶的 B-Tree 满足以下条件：

- 每个结点至多有 M 个孩子；
- 除根结点和叶结点外，其它每个结点至少有 M/2 个孩子；
- 根结点至少有两个孩子（除非该树仅包含一个结点）；
- 所有叶结点在同一层，叶结点不包含任何关键字信息；
- 有 K 个关键字的非叶结点恰好包含 K+1 个孩子；

对于任意结点，其内部的关键字 Key 是升序排列的。每个节点中都包含了 data。

![img](https://raw.githubusercontent.com/dunwu/images/master/images/database/RDB/B-TREE.png)

对于每个结点，主要包含一个关键字数组 Key[]，一个指针数组（指向儿子）Son[]。

在 B-Tree 内，查找的流程是：

1. 使用顺序查找（数组长度较短时）或折半查找方法查找 Key[]数组，若找到关键字 K，则返回该结点的地址及 K 在 Key[]中的位置；
2. 否则，可确定 K 在某个 Key[i]和 Key[i+1]之间，则从 Son[i]所指的子结点继续查找，直到在某结点中查找成功；
3. 或直至找到叶结点且叶结点中的查找仍不成功时，查找过程失败。

#### 2.1.2. B+Tree

B+Tree 是 B-Tree 的变种：

- 每个节点的指针上限为 2d 而不是 2d+1（d 为节点的出度）。
- 非叶子节点不存储 data，只存储 key；叶子节点不存储指针。

![img](https://raw.githubusercontent.com/dunwu/images/master/images/database/RDB/B+TREE.png)

由于并不是所有节点都具有相同的域，因此 B+Tree 中叶节点和内节点一般大小不同。这点与 B-Tree 不同，虽然 B-Tree 中不同节点存放的 key 和指针可能数量不一致，但是每个节点的域和上限是一致的，所以在实现中 B-Tree 往往对每个节点申请同等大小的空间。

##### 带有顺序访问指针的 B+Tree

一般在数据库系统或文件系统中使用的 B+Tree 结构都在经典 B+Tree 的基础上进行了优化，增加了顺序访问指针。

![img](https://raw.githubusercontent.com/dunwu/images/master/images/database/RDB/%E5%B8%A6%E6%9C%89%E9%A1%BA%E5%BA%8F%E8%AE%BF%E9%97%AE%E6%8C%87%E9%92%88%E7%9A%84B+Tree.png)

在 B+Tree 的每个叶子节点增加一个指向相邻叶子节点的指针，就形成了带有顺序访问指针的 B+Tree。

这个优化的目的是为了提高区间访问的性能，例如上图中如果要查询 key 为从 18 到 49 的所有数据记录，当找到 18 后，只需顺着节点和指针顺序遍历就可以一次性访问到所有数据节点，极大提到了区间查询效率。

#### 2.1.3. Hash

Hash 索引只有精确匹配索引所有列的查询才有效。

对于每一行数据，对所有的索引列计算一个 hashcode。哈希索引将所有的 hashcode 存储在索引中，同时在 Hash 表中保存指向每个数据行的指针。

哈希索引的优点：

- 因为索引数据结构紧凑，所以查询速度非常快。

哈希索引的缺点：

- 哈希索引数据不是按照索引值顺序存储的，所以无法用于排序。
- 哈希索引不支持部分索引匹配查找。如，在数据列 (A,B) 上建立哈希索引，如果查询只有数据列 A，无法使用该索引。
- 哈希索引只支持等值比较查询，不支持任何范围查询，如 WHERE price > 100。
- 哈希索引有可能出现哈希冲突，出现哈希冲突时，必须遍历链表中所有的行指针，逐行比较，直到找到符合条件的行。

### 2.2. B-Tree 和 B+Tree 有什么区别？

- B+Tree 更适合外部存储(一般指磁盘存储)，由于内节点(非叶子节点)不存储 data，所以一个节点可以存储更多的内节点，每个节点能索引的范围更大更精确。也就是说使用 B+Tree 单次磁盘 IO 的信息量相比较 B-Tree 更大，IO 效率更高。
- mysql 是关系型数据库，经常会按照区间来访问某个索引列，B+Tree 的叶子节点间按顺序建立了链指针，加强了区间访问性，所以 B+Tree 对索引列上的区间范围查询很友好。而 B-Tree 每个节点的 key 和 data 在一起，无法进行区间查找。

### 2.3. 索引原则有哪些？

#### 2.3.1. 独立的列

如果查询中的列不是独立的列，则数据库不会使用索引。

“独立的列” 是指索引列不能是表达式的一部分，也不能是函数的参数。

❌ 错误示例：

```
SELECT actor_id FROM sakila.actor WHERE actor_id + 1 = 5;
SELECT ... WHERE TO_DAYS(CURRENT_DAT) - TO_DAYS(date_col) <= 10;
```

#### 2.3.2. 前缀索引和索引选择性

有时候需要索引很长的字符列，这会让索引变得大且慢。

解决方法是：可以索引开始的部分字符，这样可以大大节约索引空间，从而提高索引效率。但这样也会降低索引的选择性。

索引的选择性是指：不重复的索引值和数据表记录总数的比值。最大值为 1，此时每个记录都有唯一的索引与其对应。选择性越高，查询效率也越高。

对于 BLOB/TEXT/VARCHAR 这种文本类型的列，必须使用前缀索引，因为数据库往往不允许索引这些列的完整长度。

要选择足够长的前缀以保证较高的选择性，同时又不能太长（节约空间）。

#### 2.3.3. 多列索引

不要为每个列创建独立的索引。

#### 2.3.4. 选择合适的索引列顺序

经验法则：将选择性高的列或基数大的列优先排在多列索引最前列。

但有时，也需要考虑 WHERE 子句中的排序、分组和范围条件等因素，这些因素也会对查询性能造成较大影响。

#### 2.3.5. 聚簇索引

聚簇索引不是一种单独的索引类型，而是一种数据存储方式。

聚簇表示数据行和相邻的键值紧凑地存储在一起。因为无法同时把数据行存放在两个不同的地方，所以一个表只能有一个聚簇索引。

#### 2.3.6. 覆盖索引

索引包含所有需要查询的字段的值。

具有以下优点：

- 因为索引条目通常远小于数据行的大小，所以若只读取索引，能大大减少数据访问量。
- 一些存储引擎（例如 MyISAM）在内存中只缓存索引，而数据依赖于操作系统来缓存。因此，只访问索引可以不使用系统调用（通常比较费时）。
- 对于 InnoDB 引擎，若辅助索引能够覆盖查询，则无需访问主索引。

#### 2.3.7. 使用索引扫描来做排序

索引最好既满足排序，又用于查找行。这样，就可以使用索引来对结果排序。

#### 2.3.8. = 和 in 可以乱序

比如 a = 1 and b = 2 and c = 3 建立（a,b,c）索引可以任意顺序，mysql 的查询优化器会帮你优化成索引可以识别的形式。

#### 2.3.9. 尽量的扩展索引，不要新建索引

比如表中已经有 a 的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可。

#### 2.3.10.如何维护索引

注意事项

- 索引不是越多越好，过多的索引不但会降低写效率，而且会降低读的效率

- 定期维护索引碎片

- 在SQL语句中不要使用强制索引关键字

  ```Sql
  FORCE INDEX
  
  SELECT * FROM TABLE FORCE INDEX (FIELD1)
  ```

  

## 3. 事务

### 3.1. 数据库事务隔离级别？事务隔离级别分别解决什么问题？

- 未提交读（READ UNCOMMITTED） - 事务中的修改，即使没有提交，对其它事务也是可见的。
- 提交读（READ COMMITTED） - 一个事务只能读取已经提交的事务所做的修改。换句话说，一个事务所做的修改在提交之前对其它事务是不可见的。
- 可重复读（REPEATABLE READ） - 保证在同一个事务中多次读取同样数据的结果是一样的。
- 可串行化（SERIALIXABLE） - 强制事务串行执行。

| 隔离级别 | 脏读 | 不可重复读 | 幻影读 |
| :------: | :--: | :--------: | :----: |
| 未提交读 | YES  |    YES     |  YES   |
|  提交读  |  NO  |    YES     |  YES   |
| 可重复读 |  NO  |     NO     |  YES   |
| 可串行化 |  NO  |     NO     |   NO   |

### 3.2. 如何解决分布式事务？若出现网络问题或宕机问题，如何解决？

## 4. 锁

### 4.1. 数据库锁有哪些类型？如何实现？

#### 4.1.1. 锁粒度

- **表级锁（table lock）** - 锁定整张表。用户对表进行写操作前，需要先获得写锁，这会阻塞其他用户对该表的所有读写操作。只有没有写锁时，其他用户才能获得读锁，读锁之间不会相互阻塞。
- **行级锁（row lock）** - 仅对指定的行记录进行加锁，这样其它进程还是可以对同一个表中的其它记录进行操作。

InnoDB 行锁是通过给索引上的索引项加锁来实现的。只有通过索引条件检索数据，InnoDB 才使用行级锁；否则，InnoDB 将使用表锁！

索引分为主键索引和非主键索引两种，如果一条 sql 语句操作了主键索引，MySQL 就会锁定这条主键索引；如果一条语句操作了非主键索引，MySQL 会先锁定该非主键索引，再锁定相关的主键索引。在 UPDATE、DELETE 操作时，MySQL 不仅锁定 WHERE 条件扫描过的所有索引记录，而且会锁定相邻的键值，即所谓的 next-key locking。

当两个事务同时执行，一个锁住了主键索引，在等待其他相关索引。另一个锁定了非主键索引，在等待主键索引。这样就会发生死锁。发生死锁后，InnoDB 一般都可以检测到，并使一个事务释放锁回退，另一个获取锁完成事务。

#### 4.1.2. 读写锁

- 排它锁（Exclusive），简写为 X 锁，又称写锁。
- 共享锁（Shared），简写为 S 锁，又称读锁。

有以下两个规定：

- 一个事务对数据对象 A 加了 X 锁，就可以对 A 进行读取和更新。加锁期间其它事务不能对 A 加任何锁。
- 一个事务对数据对象 A 加了 S 锁，可以对 A 进行读取操作，但是不能进行更新操作。加锁期间其它事务能对 A 加 S 锁，但是不能加 X 锁。

锁的兼容关系如下：

|  -   |  X   |  S   |
| :--: | :--: | :--: |
|  X   |  NO  |  NO  |
|  S   |  NO  | YES  |

使用：

- 排他锁：`SELECT ... FOR UPDATE;`
- 共享锁：`SELECT ... LOCK IN SHARE MODE;`

innodb 下的记录锁（也叫行锁），间隙锁，next-key 锁统统属于排他锁。

在 InnoDB 中，行锁是通过给索引上的索引项加锁来实现的。如果没有索引，InnoDB 将会通过隐藏的聚簇索引来对记录加锁。另外，根据针对 sql 语句检索条件的不同，加锁又有以下三种情形需要我们掌握。

1. Record lock：对索引项加锁。若没有索引项则使用表锁。
2. Gap lock：对索引项之间的间隙加锁。
3. Next-key lock：1+2，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题。当利用范围条件而不是相等条件获取排他锁时，innoDB 会给符合条件的所有数据加锁。对于在条件范围内但是不存在的记录，叫做间隙。innoDB 也会对这个间隙进行加锁。另外，使用相等的检索条件时，若指定了本身不存在的记录作为检索条件的值的话，则此值对应的索引项也会加锁。

#### 4.1.3. 意向锁

使用意向锁（Intention Locks）可以更容易地支持多粒度封锁。

在存在行级锁和表级锁的情况下，事务 T 想要对表 A 加 X 锁，就需要先检测是否有其它事务对表 A 或者表 A 中的任意一行加了锁，那么就需要对表 A 的每一行都检测一次，这是非常耗时的。

意向锁在原来的 X/S 锁之上引入了 IX/IS，IX/IS 都是表锁，用来表示一个事务想要在表中的某个数据行上加 X 锁或 S 锁。有以下两个规定：

- 一个事务在获得某个数据行对象的 S 锁之前，必须先获得表的 IS 锁或者更强的锁；
- 一个事务在获得某个数据行对象的 X 锁之前，必须先获得表的 IX 锁。

通过引入意向锁，事务 T 想要对表 A 加 X 锁，只需要先检测是否有其它事务对表 A 加了 X/IX/S/IS 锁，如果加了就表示有其它事务正在使用这个表或者表中某一行的锁，因此事务 T 加 X 锁失败。

各种锁的兼容关系如下：

|  -   |  X   |  IX  |  S   |  IS  |
| :--: | :--: | :--: | :--: | :--: |
|  X   |  NO  |  NO  |  NO  |  NO  |
|  IX  |  NO  | YES  |  NO  | YES  |
|  S   |  NO  |  NO  | YES  | YES  |
|  IS  |  NO  | YES  | YES  | YES  |

解释如下：

- 任意 IS/IX 锁之间都是兼容的，因为它们只是表示想要对表加锁，而不是真正加锁；
- S 锁只与 S 锁和 IS 锁兼容，也就是说事务 T 想要对数据行加 S 锁，其它事务可以已经获得对表或者表中的行的 S 锁。

意向锁是 InnoDB 自动加的，不需要用户干预。

## 5. 分库分表

### 5.1. 为什么要分库分表？

分库分表的基本思想就要把一个数据库切分成多个部分放到不同的数据库(server)上，从而缓解单一数据库的性能问题。

### 5.2. 分库分表的常见问题以及解决方案？

#### 5.2.1. 事务问题

方案一：使用分布式事务

- 优点：交由数据库管理，简单有效
- 缺点：性能代价高，特别是 shard 越来越多时

方案二：由应用程序和数据库共同控制

- 原理：将一个跨多个数据库的分布式事务分拆成多个仅处于单个数据库上面的小事务，并通过应用程序来总控各个小事务。
- 优点：性能上有优势
- 缺点：需要应用程序在事务控制上做灵活设计。如果使用了 spring 的事务管理，改动起来会面临一定的困难。

#### 5.2.2. 跨节点 Join 的问题

只要是进行切分，跨节点 Join 的问题是不可避免的。但是良好的设计和切分却可以减少此类情况的发生。解决这一问题的普遍做法是分两次查询实现。在第一次查询的结果集中找出关联数据的 id，根据这些 id 发起第二次请求得到关联数据。

#### 5.2.3. 跨节点的 count,order by,group by 以及聚合函数问题

这些是一类问题，因为它们都需要基于全部数据集合进行计算。多数的代理都不会自动处理合并工作。

解决方案：与解决跨节点 join 问题的类似，分别在各个节点上得到结果后在应用程序端进行合并。和 join 不同的是每个节点的查询可以并行执行，因此很多时候它的速度要比单一大表快很多。但如果结果集很大，对应用程序内存的消耗是一个问题。

#### 5.2.4. ID 唯一性

一旦数据库被切分到多个物理节点上，我们将不能再依赖数据库自身的主键生成机制。一方面，某个分区数据库自生成的 ID 无法保证在全局上是唯一的；另一方面，应用程序在插入数据之前需要先获得 ID，以便进行 SQL 路由。

一些常见的主键生成策略：

- 使用全局唯一 ID：GUID。
- 为每个分片指定一个 ID 范围。
- 分布式 ID 生成器 (如 Twitter 的 Snowflake 算法)。

#### 5.2.5. 数据迁移，容量规划，扩容等问题

来自淘宝综合业务平台团队，它利用对 2 的倍数取余具有向前兼容的特性（如对 4 取余得 1 的数对 2 取余也是 1）来分配数据，避免了行级别的数据迁移，但是依然需要进行表级别的迁移，同时对扩容规模和分表数量都有限制。总得来说，这些方案都不是十分的理想，多多少少都存在一些缺点，这也从一个侧面反映出了 Sharding 扩容的难度。

#### 5.2.6. 分库数量

分库数量首先和单库能处理的记录数有关，一般来说，Mysql 单库超过 5000 万条记录，Oracle 单库超过 1 亿条记录，DB 压力就很大(当然处理能力和字段数量/访问模式/记录长度有进一步关系)。

#### 5.2.7. 跨分片的排序分页

- 如果是在前台应用提供分页，则限定用户只能看前面 n 页，这个限制在业务上也是合理的，一般看后面的分页意义不大（如果一定要看，可以要求用户缩小范围重新查询）。
- 如果是后台批处理任务要求分批获取数据，则可以加大 page size，比如每次获取 5000 条记录，有效减少分页数（当然离线访问一般走备库，避免冲击主库）。
- 分库设计时，一般还有配套大数据平台汇总所有分库的记录，有些分页查询可以考虑走大数据平台。

### 5.3. 如何设计可以动态扩容缩容的分库分表方案？

### 5.4. 有哪些分库分表中间件？各自有什么优缺点？底层实现原理？

#### 简单易用的组件：

- [当当 sharding-jdbc](https://github.com/dangdangdotcom/sharding-jdbc)
- [蘑菇街 TSharding](https://github.com/baihui212/tsharding)

#### 强悍重量级的中间件：

- [sharding](https://github.com/go-pg/sharding)
- [TDDL Smart Client 的方式（淘宝）](https://github.com/alibaba/tb_tddl)
- [Atlas(Qihoo 360)](https://github.com/Qihoo360/Atlas)
- [alibaba.cobar(是阿里巴巴（B2B）部门开发)](https://github.com/alibaba/cobar)
- [MyCAT（基于阿里开源的 Cobar 产品而研发）](http://www.mycat.org.cn/)
- [Oceanus(58 同城数据库中间件)](https://github.com/58code/Oceanus)
- [OneProxy(支付宝首席架构师楼方鑫开发)](http://www.cnblogs.com/youge-OneSQL/articles/4208583.html)
- [vitess（谷歌开发的数据库中间件）](https://github.com/youtube/vitess)

## 6. 数据库优化

### 6.1. 什么是执行计划？

## 7. 数据库架构设计

### 7.1. 高并发系统数据层面如何设计？

#### 读写分离的原理

主服务器用来处理写操作以及实时性要求比较高的读操作，而从服务器用来处理读操作。

读写分离常用代理方式来实现，代理服务器接收应用层传来的读写请求，然后决定转发到哪个服务器。

MySQL 读写分离能提高性能的原因在于：

- 主从服务器负责各自的读和写，极大程度缓解了锁的争用；
- 从服务器可以配置 MyISAM 引擎，提升查询性能以及节约系统开销；
- 增加冗余，提高可用性。

![img](https://raw.githubusercontent.com/dunwu/images/master/images/database/mysql/master-slave-proxy.png)

##### Mysql 的复制原理

Mysql 支持两种复制：基于行的复制和基于语句的复制。

这两种方式都是在主库上记录二进制日志，然后在从库重放日志的方式来实现异步的数据复制。这意味着：复制过程存在时延，这段时间内，主从数据可能不一致。

主要涉及三个线程：binlog 线程、I/O 线程和 SQL 线程。

- **binlog 线程** ：负责将主服务器上的数据更改写入二进制文件（binlog）中。
- **I/O 线程** ：负责从主服务器上读取二进制日志文件，并写入从服务器的中继日志中。
- **SQL 线程** ：负责读取中继日志并重放其中的 SQL 语句。

![img](https://raw.githubusercontent.com/dunwu/images/master/images/database/mysql/master-slave.png)

#### 垂直切分

按照业务线或功能模块拆分为不同数据库。

更进一步是服务化改造，将强耦合的系统拆分为多个服务。

#### 水平切分

- 哈希取模：hash(key) % NUM_DB
- 范围：可以是 ID 范围也可以是时间范围
- 映射表：使用单独的一个数据库来存储映射关系

http://www.netingcn.com/mac-os-plist.html)

