## 总结

1. 页是 MySQL 中磁盘和内存交互的基本单位，也是 MySQL 是管理存储空间的基本单位，一个页一般是 16KB ，当记录中的数据太多，当前页放不下的时候，会把多余的数据存储到其他页中，这种现象称为 行溢出 。 

InnoDB 目前定义了4种行格式

* COMPACT行格式
  <img src="/Users/sh00050ml/Desktop/typora/mysql/mysql是怎样运行的/pic/c4_compact.png" alt="截屏2022-09-08 下午4.03.47" style="zoom: 33%;" />
* Redundant行格式(mysql5.0之前使用的)
  <img src="/Users/sh00050ml/Library/Application Support/typora-user-images/c4_Redundant.png" alt="截屏2022-09-08 下午4.05.52" style="zoom:33%;" />
* Dynamic行格式(5.7默认)：区别于 COMPACT行格式，在记录的真实数据处存储的是其他页的地址。
* Compressed行格式：区别于Dynamic行格式，会采用压缩算法对页面进行压缩。



## Innodb记录结构

Innodb采用的是 以页作为磁盘和内存之间交互的基本单位，Innodb中页的大小一般为16k，也就是16384个字节。

Innodb存储引擎会默认给每行记录添加2或者3个隐藏列：

DB_ROW_ID[可选]：行id，唯一标识一行

DB_TRX_ID：事务id

DB_ROLL_PTR：回滚指针

```sql
InnoDB 表对主键的生成策略：优先使用用户自定义主键作为主键，如果用户没有定义主键，则选取一个 Unique 键作为主键，如果表中连 Unique 键都没有定义的话，则 InnoDB 会为表默认添加一个名为row_id 的隐藏列作为主键。也即：InnoDB存储引擎会为每条记录都添加transaction_id和 roll_pointer 这两个列，但是 row_id 是可选的（在没有自定义主键以及Unique键的情况下才会添加该列）。
```

###  VARCHAR(M)最多能存储的数据

以ascii 字符集为例（一个字符就代表一个字节）：

Varchar 类型的列，需要占用3部分空间：

1. 真实数据
2. 真实数据占用的字节长度(占用2字节空间)
3. NULL 值标识，如果该列有 NOT NULL 属性则可以没有这部分存储空间（占用1字节空间）

综上，varchar类型的列最多能存放65533个字节的数据（有not null 标识，如果没有not null标识，就是65532个）

同理， gbk 字符集表示一个字符最多需要 2 个字节，那在该字符集下， M 的最大取值就是 32766 （也就是：65532/2），也就是说最多能存储 32766 个字符；utf8 字符集表示一个字符最多需要 3 个字节，那在该字符集下， M 的最大取值就是 21844 ，就是说最多能存储 21844 （也就是：65532/3）个字符。



```sql
注意：上述情况说的是一行数据只有一列的情况下，一定要记住一个行中的所有列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过65535个字节！
```

### 行溢出数据

在 Compact 和 Reduntant 行格式中，

varchar最多能存放65533个字节，一页放不下（16k，16843字节），会分散存储。然后会在**记录的真实数据尾部追加**使用20个字节来保存分散后的数据的地址和占用长度。

在Dynamic 行格式中，

记录的真实数据处没有数据，全部存储的是其它页的地址。

Compressed 行格式和 Dynamic 不同的一点是， Compressed 行格式会采用压缩算法对页面进行压缩，以节省空间。



被溢出的页数据，称为溢出页。

可能发生行溢出的列：varchar、text、blob

#### 溢出公式

每页需要占用的空间为136字节，每行记录占用的空间是27字节：

* 2个字节用于存储真实数据的长度

* 1个字节用于存储列是否是NULL值 

* 5个字节大小的头信息

* 6个字节的 row_id 列 

* 6个字节的 transaction_id 列 

* 7个字节的 roll_pointer 列

mysql规定一页至少存放两行记录，如果每行只有一列数据，那么行溢出的计算公式为：

136 + 2×(27 + n) > 16384

n > 8098

多列n更小。



