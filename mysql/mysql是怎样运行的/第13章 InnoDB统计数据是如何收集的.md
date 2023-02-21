我们之前使用到的 show table status 语句可以看到关于表的统计数据，通过 show index 语句可以看到关于索引的统计数据。

本章聚焦于InnoDB存储引擎的统计数据手机策略。

# 统计数据的存储方式

InnoDB提供了两种存储统计数据的方式：

* 永久性的存储统计数据：统计数据存储在磁盘上，在服务器重启后，这些统计数据依然存在。
* 非永久性的存储统计数据：统计数据存储在内存中，当服务器关闭时这些统计数据就会被清除掉。等到服务器重启后，在某些场景下会重新收集这些统计数据。

mysql 5.6.6之前，系统变量innodb_stats_persistent值默认为OFF，意思就是统计数据默认存储在内存中，在之后的版本中，innodb_stats_persistent的值默认为ON，也就是统计数据默认存储在磁盘上。

我们可以在创建表或修改表的时候，通过指定stats_persistent属性来指明该表的统计数据的存储方式：

create table table_name (...) Engine=InnoDB,stats_persistent=(1|0);

alter table table_name Engine=InnoDB,stats_persistent=(1|0);

1代表磁盘，0代表内存，不写就会使用innodb_stats_persistent的默认值。

# 基于磁盘的永久性统计数据

当我们选择把某个表以及该表的索引的统计数据放到磁盘上时，实际上是把这些统计数据放到了两个表中：

Show tables from mysql like 'innodb%stats'.

* Innodb_table_stats：存储了关于表的统计数据，每一条记录对应着一个表的统计数据；
* innodb_index_stats：存储了关于索引的统计数据，每一条记录对应着一个索引的一个统计项的统计数据。

具体字段对应的意思，用到再查吧。

# 基于内存的非永久性统计数据

后续版本不再使用这种方式，就不再记录了。



# 总结

InnoDB以表为单位来手机统计数据。这些统计数据可以是基于磁盘的永久性统计数据，也可以是基于内存的非永久性统计数据。

mysql 5.6.6之前，系统变量innodb_stats_persistent值默认为OFF，意思就是统计数据默认存储在内存中，在之后的版本中，innodb_stats_persistent的值默认为ON，也就是统计数据默认存储在磁盘上。

innodb_stats_persistent_ sample_pages 控制着永久性统计数据的采样页面数量 

innodb_ stats _ transient_sample_pages 控制着非永久性统计数据的采样页面数量 

innodb_stats_ auto_recalc 控制着是否自动重新计算统计数据.

我们可以在创建和修改表时通过指定 STATS_RSISTENT, STATS_AUTO_RECALC,STATS_SAMPLE_PAGES 的值来控制收集统计数据时的一些细节.

Innodb_stats_method决定着在统计某个索引列中不重复的值的数量时，如何对待null值。