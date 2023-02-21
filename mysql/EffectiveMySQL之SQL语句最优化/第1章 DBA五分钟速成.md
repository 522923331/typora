# 1.1识别性能问题

## 1.1.1 寻找运行缓慢的sql语句

```sql
show full processlist
```

1. 通过重复执行sql语句并记录执行时间，来判断是否是低效查询。
2. 生成执行计划，分析sql

优化sql绝不仅仅是添加索引，还可以通过优化应用程序代码来解决问题。

show create table 可以查询表结构信息

show table status like '表名'：可以查看表信息

