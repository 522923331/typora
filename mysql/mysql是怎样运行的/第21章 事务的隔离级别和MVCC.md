# 事务隔离级别

mysql是一个客户端/服务器架构的软件，对于同一个服务器来说，可以有多个客户端与之连接。那么就会存在多个请求处理同一条数据的并发情况。

## 事务并发执行时遇到的一致性问题

在读写或者写读的情况下，会出现脏读、不可重复读和幻读现象。

* 脏写（Dirty Write）：如果一个事务修改了另外一个未提交事务修改过的数据，就意味着发生了脏写现象。

* 脏读（Dirty Read）：如果一个事务读到了另外一个未提交事务修改过的数据，就意味着发生了脏读现象。

  也就是t1先修改了x的值，t2读取了未提交事务t1针对数据x修改后的值，之后t1中止t2提交。这就等于是t2读取了一个根本不存在的值。

* 不可重复读（Non-Repeatable Read）：如果一个事务修改了另外一个未提交事务读取的数据，就意味着发生了不可重复读现象。

  t1读取了x的值，t2修改了未提交事务t1读取的x的值，之后t2提交，然后t1再次读取数据x的值时，会得到与第一次读取时不同的值（同一个读事务中），这就是不可重复读。

* 幻读（Phantom）：如果一个事务先根据某些搜索条件查询出一些记录，在该事务未提交时，另外一个事务写入了一些符合那些搜索条件的记录，就意味着发生了幻读的现象。

  也就是t1先读取符合搜索条件P的记录，然后t2写入了符合搜索条件P的记录。之后t1再读取时，发现两次读取的记录数不一样了（肯定也是在同一个读事务中）。

## sql标准中的4种隔离级别

* READ UNCOMMITED：未提交读：可能发生脏读、不可重复读和幻读现象。
* READ COMMITED：已提交读：可能发生不可重复读和幻读现象，但是不可能发生脏读现象。
* REPEATABLE READ：可重复读：可能发生幻读现象，但是不可能发生脏读和不可重复读现象。
* SERIALIZABLE：可串行化：上述现象都不可能发生

为什么没有脏写？因为脏写对一致性影响太严重了，无论哪种隔离级别，都不允许脏写情况发生。

InnoDB使用锁来保证不会出现脏写现象。在第一个事务更新某条记录前，就会给这条记录加锁；另一个事务再次更新该记录时，需要等待锁释放。

mysql默认事务隔离级别为RR (REPEATABLE READ)。

设置事务的隔离级别：SET [GLOBAL | SESSION] TRANSACTION ISOLATION LEVEL SERIALIZABLE;

# MVCC

多版本并发控制（Multi-Version Concurrency Control，MVCC）：undo日志会通过roll_pointer属性来连成一个链表，叫版本链。每个版本中还包含生成该版本时对应的事务id。mvcc就是通过版本链来控制并发事务访问相同记录时的行为。

## ReadView

有的地方把readview翻译成一致性视图。

```sql
readview{
m_ids:在生成readview时，当前系统中活跃的读写事务的事务id列表
min_trx_id:在生成readview时，当前系统中活跃的读写事务中最小的事务id，也就是m_ids的最小值
max_trx_id:在生成readview时，系统应该分配给下一个事务的事务id值（大于当前存在的所有的事务id）
creator_trx_id:生成该readview的事务的事务id
}
```

通过readview来判断记录的某个版本是否可见：

* 如果被访问版本的trx_id属性值与readView中的creator_trx_id相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问
* 如果被访问版本的trx_id属性值小于readView中的min_trx_id值，表明生成该版本的事务在当前事务生成readView前已经提交，所以该版本可以被当前事务访问
* 如果被访问版本的trx_id属性值大于或等于readView中的max_trx_id值，表明生成该版本的事务在当前事务生成readView后才开启，所以该版本不能被当前事务访问
* 如果被访问版本的trx_id属性值在readView的min_trx_id和max_trx_id之间，则需要判断trx_id属性值是否在m_ids列表中。如果在，说明创建readView时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建readView时生成该版本的事务已经被提交，该版本可以被访问

如果某个版本的数据对当前事务不可见，那就顺着版本链找到下一个版本的数据，并继续执行上面的步骤来判断记录的可见性；如果记录直到最后一个版本也不可见，就意味着该条记录对当前事务完全不可见，查询结果就不包含该记录。

## READ COMMITED和REPEATABLE READ隔离级别的区别

在mysql中，READ COMMITED和REPEATABLE READ隔离级别之间一个非常大的区别就是它们生成readView的时机不同。

* READ COMMITED：每次读取数据前都生成一个readView

  举个例子：

  ```sql
  #使用READ COMMITED隔离级别的事务
  BEGIN;
  #此时name为刘备，有两个事务在进行中，TRANSACTION 100 更新为张飞、200更新为诸葛亮，但是这俩事务均未提交
  select * from hero where id = 1; #得到的name列的值为 刘备
  #TRANSACTION 100提交了，TRANSACTION 200未提交
  select * from hero where id = 1; #得到的name列的值为 张飞
  #TRANSACTION 100和200都提交了
  select * from hero where id = 1; #得到的name列的值为 诸葛亮
  ```

  也就是说，在同一个事务中，每次查询得到的结果可能都不一样，也就是读已提交的含义。因为每次查询开始时，都会生成一个独立的readView。

* REPEATABLE READ：在第一次读取数据时生成一个readView

  也可以使用上面的例子，由于快照是在第一次查询时就生成了，后续每次再查询时都是根据当时的快照来处理的，所以，查询的结果都是一样的，这就是可重复读的含义。

## mvcc小结

所谓mvcc指的就是在使用READ COMMITED、REPEATABLE READ这两种隔离级别的事务执行普通的select操作时，访问记录的版本链的过程。

在第20章说过，在执行delete或者update语句时，并不会立即把对应的记录完全从页面中删除，而是执行一个delete mark操作，给记录打上一个删除标识，这主要是为MVCC服务的。比如在RR级别下，两个事务t1和t2，t1先查询到一条记录，t2将其删除，然后t1再查询一遍，如果t2执行删除后将该记录彻底删除了，那么t1就查询不到这条记录了，就违背了可重复读的规则了。

另外，只有进行普通的select查询时，MVCC才生效。啥是不普通的？举例：

```sql
#事务T1使用REPEATABLE READ隔离级别的事务
BEGIN;
#此时表中并没有id=30的数据
select * from hero where id = 30; #得到的是空集合
#此时事务T2执行了：insert into hero values(30,'关羽','魏');并提交
#接着T1就开始更新id=30的数据
update hero set country = '蜀' where id = 30;
#然后再执行查询
select * from hero where id = 30; #得到了一条数据：30,关羽,蜀
```

原因就是，ReadView并不能阻止T1执行update或者delete语句来改动这个新插入的记录（由于T2已经提交，改动记录并不会造成阻塞），但是这样一来，这条新记录的trx_id隐藏列的值就变成了T1的事务id。之后T1再使用普通的select语句去查询这条记录时，就可以看到这条记录了。因为这个特殊现象的存在，我们也可以认为InnoDB中的MVCC并不能完全禁止幻读。



## 关于purge

有两个问题：

* insert undo 日志在事务提交后就可以释放了，而update undo日志由于还需要支持MVCC，因此不能删除掉
* 为了支持MVCC，delete mark操作仅仅是在记录上打上一个删除标记，并没有将记录真正的删除

也就是说，update 类型的undo 日志以及打了删除标记的记录什么时候会被删除？

undo log是按照事务提交no的顺序通过history链表来串联起来的，当一组update redo log的事务no 小于当前最早的readView的事务no时，就可以将这组redo log从history链表中移除，同时将打了删除标记的记录彻底删除。

# 总结

并发事务在运行过程中会出现一些可能引发一致性问题的现象（在读写或者写读的情况下，会出现脏读、不可重复读和幻读现象。）：

* 脏读
* 不可重复读
* 幻读

sql标准中的4种隔离级别：

* READ UNCOMMITED：未提交读：可能发生脏读、不可重复读和幻读现象。
* READ COMMITED：已提交读：可能发生不可重复读和幻读现象，但是不可能发生脏读现象。
* REPEATABLE READ：可重复读：可能发生幻读现象，但是不可能发生脏读和不可重复读现象。
* SERIALIZABLE：可串行化：上述现象都不可能发生

聚簇索引记录和undo日志中的roll_pointer属性可以串联成一个记录的版本链。

通过生成readView来判断记录的某个版本的可见性，其中READ COMMITED在每一次进行普通select操作钱都会生成一个readView，而REPEATABLE READ只在第一次进行普通select操作前生成一个readView，之后的查询都重复使用这个readView。

当前系统中，如果最早生成的readView不再访问undo日志以及打了删除标记的记录，则可以通过purge操作将它们清除。