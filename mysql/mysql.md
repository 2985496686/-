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
binlog不具备崩溃后数据恢复的作用。存储引擎更新一条记录是很慢的，每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程IO成本、查找成本都很高。如果大量更新请求到来，写完binlog后，存储引擎内部会积攒大量更新任务，此时服务器crash了，重启后数据就丢失了，如果再使用binlog去恢复表，会发现恢复后数据产生了不一致。



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
1. 先写binlog，在写redo log。如果在写redolog之前，mysql crash掉了，重启mysql数据丢失。如果使用

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTQ0NDg0MzU5OCwzMTI1Mzg3OTAsLTc5Mj
YwMDgzLDg0NzAxOTE3MiwtNjg4MzI5MDUsMTI0ODc4Mzg5OSwt
NzQ3Mjc1MTYwXX0=
-->