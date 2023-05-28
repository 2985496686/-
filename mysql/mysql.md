# 一条查询语句的执行流程

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/%E7%AC%94%E8%AE%B0/3v0uCsqsCpnDmpxF.png)

## 连接器
- 连接器负责跟客户端建立连接、获取权限、维持和管理连接。
- 户名密码认证通过，连接器会到权限表里面查出你拥有的权限。之后，这个连接里面的权限判断逻辑，都将依赖于此时读到的权限。此时如果修改了权限表，只有重新建立连接才会生效。
- MySQL在执行过程中临时使用的内存是管理在连接对象里面的。这些资源会在连接断开的时候才释放。所以建议在执行一个占用内存大的查询后重新建立连接，也可以执行``mysql_reset_connectio`` 重新初始化连接资源。


## 查询缓存
- 会将查询结果进行缓存，但是一旦表数据有更新，缓存就需要全部删除，导致缓存命中率很低。
- 大多数情况缓存弊大于利，在mysql8.0已经将这个模块删除了。

## 分析器
- 语法分析器会根据语法规则，判断你输入的这个SQL语句是否满足MySQL语法。如果你的语句不对，就会收到“You have an error in your SQL syntax”的错误提醒

## 优化器
- 判断使用哪种查询方式，使用哪一个索引，让语句执行更加高效，也是在这一过程中判断，表名或者字段是否存在。


## 执行器

- 首先会判断该用户是否有查询某个表的权限，若没有权限返回错误。
- 执行调用引擎接口打开表，并读取数据，引擎将读到的数据返回给执行器，执行器进行判断，将想要的结果加入结果集，然后继续调用引擎接口，直到遍历完所有数据。

## 课后问题
``Unknown column ‘k’ in ‘where clause`` 这是在哪阶段爆出的错误？

答案：分析器



# 一条更新语句的执行流程(binlog和redo log)

## binlog

在InnoDB引擎出现之前并没有redo log，server层会用通过binlog记录逻辑日志(age字段加1)，记录完日志后就会由存储引擎去完成更新任务。

**binlog的作用**
binlog主要用来数据的归档，主从表的复制。当某个表被误删，就可以通过binlog 的恢复数据。

**binlog的缺点**
binlog不具备崩溃后数据恢复的作用。存储引擎更新一条记录是很慢的，每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程IO成本、查找成本都很高。如果大量更新请求到来，写完binlog后，存储引擎内部会积攒大量更新任务，此时服务器crash了，重启后数据就丢失了，如果此时再使用binlog去恢复表，会发现恢复后数据产生了不一致。



## redolog

redolog是InnoDB引擎独有的日志。前面说过在磁盘中修改一个数据成本很高，所以可以借助一个"记账本"，将数据修改请求先记录到"记账本"中，在数据库闲下来的时候在更新磁盘。redo log就充当了这个记账本，记录的是物理日志(age字段修改为20)。当mysql crash掉了，只要记账本还在，重启mysql时就可以保证数据不会丢失。所以redolog拥有``crash-safe``的能力。

InnoDB的redo log是固定大小的，比如可以配置为一组4个文件，每个文件的大小是1GB，那么这块“记账本”总共就可以记录4GB的操作。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示。

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/%E7%AC%94%E8%AE%B0/vceBqF0eijccwFNu.png)

这里有两个指针，``write pos`` 和``checkpoint`` 。write pos 表示当前记录位置，checkpoint表示已经完成磁盘更新的位置，write pos到checkpoint之间是空闲位置，checkpoint 到write pos表示还未更新磁盘的位置。



## 两阶段提交

说明了binlog和redolog分别是干什么的，下面讲述两个是如何配合工作的：

1.  执行器先找引擎取ID=2这一行。ID是主键，引擎直接用树搜索找到这一行。如果ID=2这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
    
2.  执行器拿到引擎给的行数据，把这个值加上1，比如原来是N，现在就是N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
    
3.  引擎将这行新数据更新到内存中，同时将这个更新操作记录到redo log里面，此时redo log处于prepare状态。然后告知执行器执行完成了，随时可以提交事务。
    
4.  执行器生成这个操作的binlog，并把binlog写入磁盘。
    
5.  执行器调用引擎的提交事务接口，引擎把刚刚写入的redo log改成提交（commit）状态，更新完成。

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/%E7%AC%94%E8%AE%B0/UrngmsjWntP0Jphl.png)


## redo log持久化时机

redo的写操作也是是很密集的，为了减少磁盘I/O次数，mysql会先将日志写入redo log buffer，并在固定时机将日志刷到磁盘，时机如下：
1. mysql正常关闭时。
2. 当 redo log buffer 中记录的写入量大于 redo log buffer 内存空间的一半时，会触发落盘；
3.   InnoDB 的后台线程每隔 1 秒，将 redo log buffer 持久化到磁盘。
4. 每次事务提交时都将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘(这个策略可由 innodb_flush_log_at_trx_commit 参数控制)。

**innodb_flush_log_at_trx_commit参数**


1. 当设置该**参数为 0 时**，表示每次事务提交时 ，还是**将 redo log 留在 redo log buffer 中** ，该模式下在事务提交时不会主动触发写入磁盘的操作
2. 当设置该**参数为 1 时**，表示每次事务提交时，都**将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘**，这样可以保证 MySQL 异常重启之后数据不会丢失。
3. 当设置该**参数为 2 时**，表示每次事务提交时，都只是缓存在 redo log buffer 里的 redo log **写到 redo log 文件，注意写入到「 redo log 文件」并不意味着写入到了磁盘**，因为操作系统的文件系统中有个 Page Cache（Page Cache 是专门用来缓存文件数据的，所以写入「 redo log文件」意味着写入到了操作系统的文件缓存）。

## redo log删除时机
当``buffer pool`` 将脏数据刷新到磁盘后，就会将没有用的redo log删除，redo log的删除时机就是buffer pool刷新脏数据的时机，如下：
-   当 redo log 日志满了的情况下，会主动触发脏页刷新到磁盘；
-   Buffer Pool 空间不足时，需要将一部分数据页淘汰掉，如果淘汰的是脏页，需要先将脏页同步到磁盘；
-   MySQL 认为空闲时，后台线程会定期将适量的脏页刷入到磁盘；
-   MySQL 正常关闭之前，会把所有的脏页刷入到磁盘；


## mysql崩溃恢复
可以简单的将过程进行如下简化：
1. 写redo log
2. 写binlog
3. commit

- 在1之后crash，重启后日志回滚。
- 在2之后crash，重启后redolog自动提交，然后正常写磁盘。
- 在3之后crash，重启后正常写磁盘。


## 两阶段提交的必要性

假设redo log没有prepare阶段，写入redo log后就可以直接更新到磁盘，会出现以下情况：
1. 先写binlog，在写redo log。如果在写redolog之前，mysql crash掉了，重启mysql数据丢失。如果使用binlog进行数据恢复，数据出现不一致的情况。
2.  先写redo log再写bin log。如果在写完bin log之前crash，重启后会拿redolog进行数据恢复，从而导致数据不一致。

## 课后问题

定期全量备份的周期“取决于系统重要性，有的是一天一备，有的是一周一备”。那么在什么场景下，一天一备会比一周一备更有优势呢？或者说，它影响了这个数据库系统的哪个指标？

答案：采用哪种备份方式取决于系统对数据恢复时间长短的容忍程度。
通过binlog恢复数据是将执行的操作重放一遍，如果积攒了大量的binlog，恢复时间也就会相应的变长。所以只依靠binlog来恢复数据肯定是不行的，还需要定期的将数据备份。如果是一周一备的话，在数据恢复时就需要依靠上次备份的数据 + 上次备份到现在积攒的binlog。如果是一天一备，最多积攒一天的binlog，数据恢复就很快。
这里可能会有一个疑问，如果实时备份数据是不是就不需要binlog了？
确实如此，但是这是很难实现的，数据备份是一个成本很高的过程，无法做到实时备份，而binlog只需要顺序写磁盘。



# mysql 索引


## 常见的索引模型

**哈希表：**  删除节点，添加节点，搜索节点效率都很高，但是不支持范围查询。适用于定值查找。
**数序数组：** 查询效率最高，但是只使用于固定数据。
**跳表：** 查找，添加，删除效率都是logN，但是跳表高度较高，不适合于需要大量磁盘IO的场景。
**B+树：**  B+树是多叉平衡搜索树，扇出高，只需要3层左右就能存放2kw左右的数据，同样情况下跳表则需要24层左右，假设层高对应磁盘IO，那么B+树的读性能会比跳表要好，因此mysql的InnoDB选了B+树做索引。


## InnoDB的索引模型

- InnoDB使用B+树索引模型，所有数据都是存储在B+树上的。
- 每一个索引在InnoDB中都对应一个B+树。
- 如果没有主键，默认将rowId作为主键索引。

有如下建表语句：
```sql
create table T( 
	id int primary key, 
	k int not null, 
	name varchar(16), 
	index (k)
)engine=InnoDB;
```
![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/%E7%AC%94%E8%AE%B0/xpKlBDZMlPPoXmwV.png)

主键对应的B+树叶子结点直接存储的是完整的数据信息，主键索引又被称为聚簇索引。
非主建索引对应的B+树叶子节点为主键值，非主键索引被称为二级索引。


根据上面的索引结构说明，我们来讨论一个问题：**基于主键索引和普通索引的查询有什么区别？**

-   如果语句是select * from T where ID=500，即主键查询方式，则只需要搜索ID这棵B+树；
-   如果语句是select * from T where k=5，即普通索引查询方式，则需要先搜索k索引树，得到ID的值为500，再到ID索引树搜索一次。这个过程称为**回表**。

也就是说，基于非主键索引的查询需要多扫描一棵索引树。因此，我们在应用中应该尽量使用主键查询。




## 索引维护

为了维护索引的有序性，每添加一个节点就需要进行维护，插入数据有两种：

1. 递增的数据，这时只需要顺序的在节点末尾插入数据。
2. 插入非递增的数据，可能会造成页分裂，影响性能，同时降低了空间利用率。


主键使用的注意点：
1. 尽量保证主键单调递增，减少不必要的页分裂次数，提高性能。
2. 主键尽量越小越好，这样普通索引的叶子节点也就越小。


 
 ## 课后问题一

重建索引是否正确？

```sql
# 非主键
alter table T drop index k; 
alter table T add index(k);

# 主键
alter table T drop primary key; 
alter table T add primary key(id);
```

答案： 单独的删除重建索引k没有问题，但是对主键的操作有问题，不论是删除主键还是创建主键，都会将整个表重建，严重影响性能，可以使用``alter table T engine=InnoDB`` 代替这两步。



## 索引覆盖

还是以上表为例：
```sql
select id from T where k between 3 and 5
```

搜索k在3到5之间所有用户的id，常规思路：
在k索引树上找出全部符合条件的数据，然后依次回表查询对应数据。但是此处只查询id，id已经在k、索引树上包含，所以没必要回表。

**结论：由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。**

## 联合索引
有这样的情况：在一个居民信息表上，需要频繁的根据身份证号查询姓名(二者均不是主键)，此时就可以建立联合索引：
```sql
key(`id_card`,`name`)
```
这样也可以达到索引覆盖的效果，减少回表次数。

联合索引还用于``多条件查询``，如果频繁的需要根据地区和年龄查询查询符合条件的居民，就可以建立地区和年龄的联合索引。



## 最左匹配原则

联合索引不只用于上面两种情况，还可用于左边字段的查询，比如：(id_card，name)的联合索引，它可以``代替id_card索引``（不能代替name)。

所以在建立联合索引的时候就需要考虑下面两个原则：
1. 第一原则是，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的。比如：除了需要频繁根据身份证号查询名字，还需要根据身份证号查询个人的全部信息，那我们就不需要单独为身份证号建立索引。
2. 第二原则就要考虑空间了。如果不仅有根据身份证号查询个人的全部信息的需求，还需要通过name查询符合条件居民的需求，应该如何建立索引？
有两种方案，如下：
```sql
# 方案一
key(`id_card`,`name`)
key(`name`)

#方案二
key(`name`,`id_card`)
key(`id_card`)
```
两种方案虽然都能达到我们预期的效果，但是方案一更加省内存(前提是id_card比name更占内存)。



# mysql中的锁

## 全局锁

1. 对整个数据库加锁，``Flush tables with read lock``，让整个数据库处于只读状态，直到客户端断开连接或者执行``unlock tables``。

2. 全局锁使用场景：作全局备份时，防止在备份两个表的时候出现逻辑时间不一致的情况。

3. 在InnoDB出现后，支持可重复读的事务隔离级别，此时也不再需要全局锁。

4. ``set global readonly=true`` 虽然也能做到全局只读，当它并不能代替全局锁或者可重复读的事务隔离级别，原因如下：
 -- 在有些系统中，readonly的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。因此，修改global变量的方式影响面更大，不建议你使用。
 -- 客户端将库设置为readonly后，如何断开连接，设置不会自动恢复。


## 表级锁
表级锁有两种，表锁和元数据锁。表锁保证两个线程不会同时操作同一个表，保证并发安全性，但是粒度太粗，InnoDB引擎出现后被行锁代替，这里不再讲解表所，重点对元数据锁进行说明。

### 元数据锁(MDL)
- 元数据锁，metadata lock，是对一个表的元数据信息加锁。
- MDL是一个读写锁，DML操作(对表数据的增删改查)是读操作，DDL(对表结构的修改)是写操作。
- MDL不需要手动加锁，在操作时会自动加读锁或写锁。
- MDL是在执行操作的时候获取的，但是锁的释放是在事务结束之后。
- mysql的InnoDB引擎虽然保证了事务隔离性，但是也不是绝对的，事务和事务之间也会受到MDL锁的影响。一个事务执行DDL操作，获取写锁，此时另一个事务执行DML操作时就会阻塞，直到第一个事务提交。

**元数据锁存在的死锁隐患**
当一个长事务获取了MDL读锁，但是并没有及时提交事务，此时另一个事务尝试获取MDL写锁被阻塞，这导致后续事务无法再获取读锁，造成整个mysql业务的阻塞。

为了解决这个问题，比较理想的机制是，在alter table语句里面设定等待时间，如果在这个指定的等待时间里面能够拿到MDL写锁最好，拿不到也不要阻塞后面的业务语句，先放弃。之后开发人员或者DBA再通过重试命令重复这个过程。

同时未了防止获取写锁的事务长时间影响其他事务，尽量将DDL操作放在事务的最后执行。


## 行锁
1. 行锁是随InnoDB引擎出现的一个细粒度锁，取代了表锁。
2. 行锁并不是一个读写锁，它只会写写互斥，在mvcc机制的保证下，读事务只会读取以提交的内容，并不需要加锁。
3. 行锁和mdl一样，在需要时获取，在事务结束时释放(两阶段锁)。

**行锁存在的死锁隐患**

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/%E7%AC%94%E8%AE%B0/DanHWIbQwHBQHwWk.png)

如上情况，就会造成死锁。

**解决方案**
- 方案一：采用设置超时时间策略。通过设置``innodb_lock_wait_timeout``参数，默认值是50s，这也是mysql默认的方案。
- 方案二：死锁检测，在获取锁时，遍历所有线程持有锁的情况，保证自己不会产生死锁，通过参数``innodb_deadlock_detect``来进行设置。这种方案在并发度高的情况下，会严重影响性能，如果同时有1000个同时更新同一行数据，就需要死锁检测1000000次。

**如何优化死锁检测的低效问题？**

- 方案一：通过中间键限制并发数量。
- 方案二：在特殊的场景下，比如说减库存，可以通过客户端水平分表解决低效问题，每张表的库存加起来就是中库存。



## 课后问题

要删除一个表里面的前10000行数据，有以下三种方法可以做到：

-   第一种，直接执行delete from T limit 10000;
-   第二种，在一个连接中循环执行20次 delete from T limit 500;
-   第三种，在20个连接中同时执行delete from T limit 500。


 答案： 选择方案二。方案一会导致长时间持有锁，并且长事务会导致主从复制的延迟；方案三会导致锁并发冲突。



# mysql放弃索引的情况

##  对条件字段进行函数操作

如下SQL：
```sql
# score有索引

# 语句一
select *from user where FLOOR(score) > 60 # FLOOR()向下取整函数

#语句二
select *from user where score - 1 > 60 
```
以上两条语句mysql都会放弃使用索引，虽然score上有索引，但是函数操作后的结果可能已经破坏了搜索树的平衡，mysql选择放弃使用索引。


## 字段隐式类型转换

如下SQL：

```sql
# score有索引，varchar类型

# 语句一
select *from user where score > '60'

#语句二
select *from user where score > 60
```

以上两条语句，第一句会使用索引，第二句不会使用。

很明显是由于隐式类型转换产生的，那mysql是类型转换规则是怎样的？
执行下面语句：
```
select '10' > 9
```
执行结果如下：
![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/%E7%AC%94%E8%AE%B0/9PUvZ2e6VIlcDuoG.png)

**说明mysql中数字与字符串比较时，会自动将字符串转为数字进行比较**

语句一不会存在类型转换，所以会走索引。
语句二对条件字段存在类型转换，mysql认为可能会破坏树的平衡，不使用索引。

如果将score字段设置为int类型，语句一是否会走索引？
会走索引，mysql默认的是将字符串转数字，条件字段是数字，而条件是字符串，所以在搜索时会对条件进行转换类型，不会字段进行操作。

##  隐式字符编码转换

在多表联查的时候，如果两个表的编码格式不同，可能就会产生因为编码转换导致索引失效的问题。

编码转换规则： 会往比自己覆盖更广的编码方式进行转换。	

## 总结

三种情况都陈述一件事实：

**对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能。**



# RR模式下的间隙锁(Gap Lock) 

## 幻读现象

InnoDB的可重复读模式下，通过MVCC机制解决了不可重复读现象，但是还存在幻读现象。**幻读指的是在一个事务中第二次读取同一个数据时，发现多出来一条数据。** 这里的读取，不仅仅是指像``select *from t`` 这样的快照读，还包括``update t set c = 100 where id = 1``这样的当前读。


## 快照读解决幻读的方案

在MVCC机制下，保证一个事务中执行两个相同的select快照读语句，不会出现幻读情况。


## 当前读解决幻读的方案


**select  fields from t where d = 5 for update** 语句表示当前读，表示接下来要对d = 5的这行数据进行update操作，提前加锁，该语句加的锁与语句  **select fields from t where** 加的锁一致，只是提前加锁了。下面分析当前读对幻读的解决。



有如下一个表，并提供一个场景：

```sql
CREATE TABLE `t` ( 
`id` int(11) NOT NULL, 
`c` int(11) DEFAULT NULL, 
`d` int(11) DEFAULT NULL,
 PRIMARY KEY (`id`), 
 KEY `c` (`c`) ) ENGINE=InnoDB; 
 insert into t values(0,0,0),(5,5,5), (10,10,10),(15,15,15),(20,20,20),(25,25,25);
```
 
 ![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/%E7%AC%94%E8%AE%B0/tg9kGTKjjKwOdYA9.png)

按照之前的认知，update操作会将对应数据加上行锁，这里暂且认为update语句只加行锁，并且for update 当前读操作与此一致，分析会出现什么后果。


事务A中当前读了三次：
第一次结果： (5,5,5)
第二次结果：(5,5,5) (0,5,5)
第三次结果：(5,5,5) (0,5,5) (1,5,5)

多出来的两个结果就是幻读产生的。(严格的来说读出新增的数据才是幻读，也就是数据1,1,5 ，但是这里0,5,5数据的出现与幻读造成的结果基本一致，所以在这里也当成幻读)。


因为在这里session A的T3和T5时刻只是进行了简单的当前读操作，看似并没有造成太大影响，但其实不然。

binlog的写入时机是在事务提交的时候，三个事务执行完之后，binlog是以如下顺序写入的：

```sql
update t set d=5 where id=0; /*(0,0,5)*/ 
update t set c=5 where id=0; /*(0,5,5)*/ 
insert into t values(1,1,5); /*(1,1,5)*/ 
update t set c=5 where id=1; /*(1,5,5)*/ 
update t set d=100 where d=5;/*所有d=5的行，d改成100*/
```
这个语句序列，不论是拿到备库去执行，还是以后用binlog来克隆一个库，这三行的结果，都变成了 (0,5,100)、(1,5,100)和(5,5,100)，**造成了数据不一致问题**。上述事例中，session A 的T3和T5时刻只是进行了当前读操作，如果换成update语句，造成的数据不一致会更复杂。

mysql是绝对无法容忍这种数据不一致的情况发生的。上面的这些情况都是假设当前读操作只有行锁，如果按照上面的事务顺序执行sql，就会发现session B和session C在session A提交之前会发生阻塞，因为innoDB引擎下，当前读操作还会存在**间隙锁**。

## 间隙锁

在执行语句 ``select  fields from t where d = 5 for update``时，不仅会在d = 5这一行加上行锁，还会加上``间隙锁``。
|id字段| 0 | 5| 10|
|--|--|--|--|
| c字段 | 0 | 5|10|
| d字段 | 0 | 5|10|

d为5的行两边有两个空隙，(0,5) 和(5,10)，这里的间隙锁就是锁住这两个空隙，保证不会在这两个空隙中插入新的数据，并且不允许修改或插入d为5的数据，合起来锁的范围就是(0,10)。
**gap lock + record lock 加起来就保证不会因为幻读产生数据不一致的情况，合起来的锁又被称为next-key  lock。**


## RR模式仍然存在的幻读现象

``next-key lock``通过锁住一部分行，解决了因为幻读带来的数据不一致的情况，但毕竟不是锁全部数据，幻读现象仍然存在。还是以上面的表为例。

- T1时刻session A 执行``select *from t  where d = 11;``
- T2时刻session B 执行``insert  into t values（11,11,11); commit; ``
- T3时刻session A执行``insert  into t values（11,11,11);`` 就会报``ERROR 1062 (23000): Duplicate entry '11' for key 't.PRIMARY'
``

虽然仍然存在幻读现象，但是如果合理的使用 ``next-key lock`` 在RR模式下也能规避幻读现象。


# binlog和redolog持久化机制

## binlog的写入机制
- 事务在执行过程中，会将log写入binlog cache 中，在日志提交的时候再把binlog cache写到binlog文件中。
- 为了保证一个事务的binlog被一次性的，完整的，连续的写入binlog文件，系统对于每一个事务线程都会分配一片内存，参数 ``binlog_cache_size``用于控制单个线程内binlog cache所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。
- binlog写入流程如下图：
![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/%E7%AC%94%E8%AE%B0/8OFV3V0ZVqkhXChf.png)

可以看到，每个线程有自己binlog cache，但是共用同一份binlog文件。

-   图中的write，指的就是指把日志写入到文件系统的page cache，并没有把数据持久化到磁盘，所以速度比较快。
-   图中的fsync，才是将数据持久化到磁盘的操作。一般情况下，我们认为fsync才占磁盘的IOPS。

write 和fsync的时机，是由参数sync_binlog控制的：

1.  sync_binlog=0的时候，表示每次提交事务都只write，不fsync；
    
2.  sync_binlog=1的时候，表示每次提交事务都会执行fsync；
    
3.  sync_binlog=N(N>1)的时候，表示每次提交事务都write，但累积N个事务后才fsync。
    

因此，在出现IO瓶颈的场景里，将sync_binlog设置成一个比较大的值，可以提升性能。在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成0，比较常见的是将其设置为100~1000中的某个数值。

但是，将sync_binlog设置为N，对应的风险是：如果主机发生异常重启，会丢失最近N个事务的binlog日志。


## redolog的写入机制

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTg1NzY2MTgzMSwyMDIxNzI1NDk1LDE3MD
U5MjUzNzcsMTM3MTUyMTY2OSwtMTIwNzg3NTE4OSwtMTQ1NjYx
OTY3NiwtMTg4NTUzNzY1NiwtMTA4OTM3OTQyNCw2MDkwNjk2Mz
QsLTExODYzMzY3NzYsMTczMzEzMzA5OSwxNzMyMTQ0MzEsLTIz
OTQ5MzAxMywtMTMwODQwMTQ0NywtNjUxMzAxNDEsNDc2NjUyMD
A4LC01MTg3Mjk4NjMsLTg1NjQwMjI3NiwxNTQ3NzYxNDA3LDUx
Mzk5NzY4MF19
-->