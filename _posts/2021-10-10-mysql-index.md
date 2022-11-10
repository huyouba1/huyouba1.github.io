---
layout: post
title: mysql 索引
date: 2021-10-10 
tags: Mysql
---
![](/images/posts/mysql/WX20211010-185223@2x.png)

### sql 性能下降原因

- 索引失效
- 语句写的烂
- 关联查询太多join(涉及缺陷或不得已的需求)
- 服务器调优及各个参数设置（缓冲、线程数等）



```sql
create index idx_user_name  on user(name)
```



##### 索引

> 数据本身之外，数据库还维护这一个满足特定查找算法的数据结构，这些数据结构以某种方式指向数据，这样就可以在这些数据结构的基础上实现高级查找算法，这种数据结构就是索引。
>
> 一般来说索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储在磁盘上

优势：

> 1）类似于大学图书馆建立书目索引，提高数据检索的效率，降低数据库的IO成本
>
> 2）通过索引对数据进行排序，降低数据排序的成本，降低了CPU的消耗

劣势：

> 1）实际上索引也是一张表，该表保存了主键与索引字段，并指向实体表的记录，所以索引也是占用空间的

> 2）虽然索引大大提高了查询速度，同时却会降低更新表的速度，如对表进行insert、update和delete。因为更新表是，mysql不仅要保存数据，还要保存一下索引文件，每次更新添加了索引列的字段，都会调整因为更新所带来的将至变化后的索引信息

> 3）索引只是提高效率的一个因素，如果你的MySQL有大数据量的表，就需要花时间研究建立最优秀的索引，或者优化查询


### 索引分类

- 单值索引
  - 即一个索引只包含单个列，一个表可以有多个单列索引
- 唯一索引
  - 索引列的值必须唯一，但是允许有空值
- 复合索引
  - 即一个索引包含多个列
- 基本语法
  - 创建 `create [UNIQUE] INDEX [indexName] ON mytable(columnname(length));` 或者 `alter mytable ADD [UNIQUE] INDEX [indexName] ON (columnname(length));`
  - 删除 `drop INDEX [indexName] ON mytable;`
  - 查看 `show index from table_name\G`





### 有四种方式来添加数据表的索引:

```sql
alter table tbl_name ADD primary key (column_list);   #该语句添加一个主键，这意味着索引值必须是唯一的，且不能为NULL
alter table tbl_name ADD UNIQUE index_name (column_list);  #这条语句创建索引的值必须是唯一的（除了NULL外，NULL可能会多次出现）
alter table tbl_name ADD INDEX index_name (column_list);  #添加普通索引，索引值可出现多次
alter table tbl_name ADD FULLTEXT index_name (column_list); #该语句制订了索引为FULLTEXT，用于全文索引
```



### mysql索引结构

- BTree索引
- Hash索引
- full-text全文索引
- R-Tree索引