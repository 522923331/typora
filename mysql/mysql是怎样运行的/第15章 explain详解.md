mysql查询优化器在基于成本和规则对一条查询语句进行优化后，会生成一个执行计划。

这个执行计划展示了接下来执行查询和具体方式，比如多表连接的顺序是什么，采用什么访问方法来具体查询每个表等。

# 执行计划输出中各列详解

explain select 1；列字段含义如下：

|     列名      | 描述                                                         |
| :-----------: | ------------------------------------------------------------ |
|      id       | 在一个大的查询语句中，每个select关键字都对应一个唯一的id     |
|  select_type  | select 关键字对应的查询类型                                  |
|     table     | 表名                                                         |
|  partitions   | 匹配的分区信息                                               |
|     type      | 针对单表的访问方法                                           |
| possible_keys | 可能用到的索引                                               |
|      Key      | 实际使用的索引                                               |
|    key_len    | 实际使用的索引长度                                           |
|      ref      | 当使用索引列等值查询时，与索引列进行等值匹配的对象信息       |
|     rows      | 预估的需要读取的记录条数                                     |
|   filtered    | 针对预估的需要读取的记录，经过搜索条件过滤后剩余记录条数的百分比 |
|     Extra     | 一些额外的信息                                               |

## table

无论查询语句有多复杂，里面包含了多少个表，到最后也是对每个表进行单表访问。

## id

Id：是为每个select语句分配一个id，如果select了多个表，那么explain后也会有多行数据，但是他们的id是相同的。

在连接查询的执行计划中，每个表都会对应一条记录，这些记录的id列的值是相同的。出现在前面的表表示驱动表，出现在后面的表表示被驱动表。

查询优化器可能对涉及子查询的查询语句进行重写，从而转换为连接查询（当然这里指的是半连接）。那么我们可以通过执行计划的id是否相等就可以知道是否被优化了。

## select_type

* SIMPLE：表示简单的select，没有union和子查询。
* PRIMARY：对于包含union、union all或者子查询的大查询来说，它是由几个小查询组成的；其中最左边的那个查询的select_type值就是PRIMARY。
* UNION：对于包含UNION或者UNION ALL的大查询来说，它是由几个小查询组成的；其中，除了最左边的那个小查询外，其余小查询的select_type值就是UNION。
* UNION RESULT：mysql选择使用临时表来完成UNION查询的去重工作，针对该临时表的查询的select_type就是UNION RESULT。
* SUBQUERY：如果包含子查询的查询语句不能够转为对应的半连接形式，并且该子查询是不相关子查询，而且查询优化器决定采用将该子查询物化的方案来执行该子查询时，该子查询的第一个SELECT关键字代表的那个查询的select_type就是SUBQUERY。
* DEPENDENT SUBQUERY：如果包含子查询的查询语句不能够转为对应的半连接形式，并且该子查询被查询优化器转换为相关子查询的形式，则该子查询的第一个select关键字代表的那个查询的select_type就是DEPENDENT SUBQUERY。
* DEPENDENT UNION：在包含UNION或者UNION ALL的大查询中，如果各个小查询都依赖于外层查询，则除了最左边的那个查询之外，其余小查询的select_type的值就是DEPENDENT UNION。
* DERIVED：在包含派生表的查询中，如果是以物化派生表的方式执行查询，则派生表对应的子查询的select_type就是DERIVED。
* MATERIALIZED：当查询优化器在执行包含子查询的语句时，选择将子查询物化之后与外层查询进行连接查询，该子查询对应的select_type属性就是MATERIALIZED。
* 余下两个不常用，就不写了。

## type

前面唠叨过对使用InnoDB存储引擎的表进行单表访问的一些方法，完整的访问方法有system，const，eq_ref，ref，fulltext，ref_or_null,index_merge，unique_subquery,index_subquery,range,index,ALL等。

* System：当表中只有一条记录并且该表使用的存储引擎（比如MyISAM，MEMORY）的统计数据是精确的，那么对该表的访问方法就是system。
* const：根据主键或者唯一二级索引列与常数进行等值匹配时，对单表的访问方法就是const。
* eq_ref：执行连接查询时，如果被驱动表是通过主键或者不允许存储null值的唯一二级索引列等值匹配的方式进行访问的（如果是联合唯一索引，则所有的索引列都必须进行等值比较），则对该被驱动表的访问方法就是eq_ref。
* ref：当通过普通的二级索引列与常量进行等值匹配的方式来查询某个表时，对该表的访问方法就可能是ref。
* fulltext：全文索引
* ref_or_null：当对普通二级索引列进行等值匹配且该索引列的值也可以是null值时，对该表的访问方法就可能是ref_or_null。
* index_merge：一般情况下，只会为单个索引生成扫描区间。就是Intersection、Union和Sort-Union这3种索引合并的方式。
* unique_subquery：类似于两表连接中被驱动表的eq_ref访问方法，unique_subquery针对的是一些包含IN子查询的查询语句。如果查询优化器决定将IN子查询转换为EXISTS子查询，而且子查询在转换后可以使用主键或者不允许存储null值的唯一二级索引进行等值匹配，那么该子查询执行计划的type列的值就是unique_subquery。
* index_subquery：index_subquery和unique_subquery类似，只不过在访问子查询中的表时使用的是普通的索引。
* range：如果使用索引获取某些单点扫描区间的记录，或者用于获取某个或者某些范围扫描区间的记录的查询，那么就可能使用到range访问方法。
* index：当可以使用索引覆盖，但需要扫描全部的索引记录时，该表的访问方法就是index。比如：explain select key_part2 from s1 where key_part3='a'; 其中s1表中有联合索引idx_key_part(key_part1,key_part2,key_part3)。
* ALL：全表扫描。

一般来说，这些访问方法的性能按照上面的顺序依次变差（当然不是绝对的，还取决于需要访问的记录的数量）。

## possible_keys和key

possible_keys：指的是执行单表查询时可能用到的索引有哪些；

Key：表示实际用到的索引有哪些

注意：possible_keys列中的值并不是越多越好，可以使用的索引列越多，查询优化器在计算查询成本时花费的时间就越长。如果可以的话，尽量删除那些用不到的索引。

## key_len

用来确定扫描区间，以及形成该扫描区间的边界条件是什么。

varchar字段的key_len值303，如果是两个varchar，则是606。

## ref

当访问方法是const、eq_ref、ref、ref_or_null、unique_subquery、index_subquery中的其中一个时，ref列展示的就是与索引列进行等值匹配的东西是啥，比如只是一个常数或者是某个列。

## rows

根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数。

## filtered

对于单表查询来说，这个字段列的值没什么意义，主要用在多表连接查询中，用来判断被驱动表大约会被执行多少次，列的数据是百分比的意思。

比如查询驱动表的rows=9888，filtered=10.0，那么就表示被驱动表将会被执行大约988次。

## Extra

用来说明一下额外信息。挑几个常见的：

* No tables used：当查询语句中没有from子句时将会提示该额外信息，如 explain select 1;
* Impossible WHERE：查询语句的where子句永远为false时将会提示该额外信息，如：explain select * from s1 where 1!=1
* No matching min/max row：当查询列表处有MIN或者MAX聚集函数，但是并没有记录符合where子句中的搜索条件时，将会提示该额外信息，如：explain select min(key1) from s1 where key1 = 'abc';
* Using index：使用覆盖索引执行查询时，Extra列将会提示该额外信息，如：explain select key1 from s1 where key1 = 'a'
* Using index condition：如果在查询语句的执行过程中使用索引条件下推特性，在Extra列中将会显示Using index condition。
* using join buffer(Block Nested Loop)：在连接查询的执行过程中，当被驱动表不能有效地利用索引加快访问速度时，mysql会为其分配一块名为连接缓冲区(join buffer)的内存块来加快查询速度，也就是使用基于块的嵌套循环算法来执行连接查询。
* Using intersect、Using union、Using sort_union：分别对应的就是使用Intersection索引合并方式、Union索引合并方式、Sort-Union索引合并方式执行查询。
* Zero limit：当limit子句的参数为0时，表示压根不打算从表中读出任何记录，此时将会提示该额外信息，如：explain select * from s1 limit 0;
* Using filesort：在有些情况下，当对结果集中的记录进行排序时，是可以使用到索引的。排序操作无法使用到索引，只能在内存中（记录较少时）或者磁盘中（记录较多时）进行排序。把这种在内存中或者在磁盘中进行排序的方式统称为文件排序（filesort）。如：explain select * from s1 order by common_field limit 10;
* Using temporary：如果查询中使用到了临时表，Extra会显示为Using temporary。场景：比如我们在执行许多包含distinct、group by、union等子句的查询过程中，如果不能有效利用索引来完成查询，就会通过建立内部临时表来执行查询。如：explain select  distinct common_field from s1；

# json格式的执行计划

在explain单词和真正的查询语句中间加上format=json

explain format=json select * from s1 inner join s2 on s1.key1 = s2.key2 where s1.common_field = 'a';

s1表的cost部分如下：

"cost_info":{

​	"read_cost": "1840.84",

​	"eval_cost": "193.76",

​	"prefix_cost": "2034.60",

​	"data_read_per_join": "1M"

}

其中，

read_cost = I/O成本+ 检测 rows  * （1-filter）条记录的CPU成本

eval_cost = 检测 rows  * filter条记录的成本

Prefix_cost 就是单独查询s1表的成本，也就是 = read_cost + eval_cost。

对于s2表的cost_info部分：

"cost_info":{

​	"read_cost": "968.80",

​	"eval_cost": "193.76",

​	"prefix_cost": "3197.16",

​	"data_read_per_join": "1M"

}

s2表的prefix_cost 代表的是整个连接查询预计的成本，也就是单次查询s1表和多次查询s2表后的成本的和，即：

968.80 + 193.76 + 2034.60 = 3197.16



# 总结

通过explain语句可以查看某个语句的执行计划。

在explain单词和真正的查询语句中间加上 format=json，可以得到json格式的执行计划

在使用explain语句查看了某个查询的执行计划后，紧接着还可以使用show warnings语句查看与这个查询的执行计划有关的扩展信息（message中展示的信息并不是标准的查询语句，只能作为我们理解mysql如何执行查询语句的一个参考依据）