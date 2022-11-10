---
layout: post
title: mysql 基础查询
date: 2021-10-10 
tags: Mysql
---
### 环境准备
```
mysql> create table IT_salary (岗位类别 char(20) not null,姓名 char(20) not null,年龄 int,员工ID int not null,学历 char(6),年限 int,薪资 int not null, primary key (员工ID));	

#int 数字类型、char 字符串类型、not null 不能为空、char（）指定最多字节个数、primary key（）指定索引字段

Query OK, 0 rows affected (0.13 sec)

mysql> desc IT_salary;

+--------------+----------+------+-----+---------+-------+
| Field        | Type     | Null | Key | Default | Extra |
+--------------+----------+------+-----+---------+-------+
| 岗位类别     | char(20) | NO   |     | NULL    |       |
| 姓名         | char(20) | NO   |     | NULL    |       |
| 年龄         | int(11)  | YES  |     | NULL    |       |
| 员工ID       | int(11)  | NO   | PRI | NULL    |       |
| 学历         | char(6)  | YES  |     | NULL    |       |
| 年限         | int(11)  | YES  |     | NULL    |       |
| 薪资         | int(11)  | NO   |     | NULL    |       |
+--------------+----------+------+-----+---------+-------+
7 rows in set (0.00 sec)

mysql> insert into IT_salary (岗位类别,姓名,年龄,员工ID,学历,年限,薪资) values ('网络工程师','孙空武',27,011,'本科',3,4800);

Query OK, 1 row affected (0.01 sec)

mysql> insert into IT_salary(岗位类别,姓名,年龄,员工ID,学历,年限,薪资) values('Windows 工程师','蓝 凌 ',19,012,'中专',2,3500);

Query OK, 1 row affected (0.00 sec)

mysql> insert into IT_salary(岗位类别,姓名,年龄,员工ID,学历,年限,薪资) values('Linux 工程师','姜纹 ',32,013,'本科',8,15000);

Query OK, 1 row affected (0.00 sec)

mysql> insert into IT_salary(岗位类别,姓名,年龄,员工ID,学历,年限,薪资) values('Java 软件工程师','   园' 38,014,'大专',10,16000);

Query OK, 1 row affected (0.00 sec)

mysql> insert into IT_salary(岗位类别,姓名,年龄,员工ID,学历,年限,薪资) values('硬件驱动工程师','罗 中  昆',29,015,'大专',9,16500);

Query OK, 1 row affected (0.01 sec)
```



### mysql 常见命令

```
# 查看库
show databases;  

# 查看表
use test;
show tables;
show tables from mysql;

# 查看当前在哪个库
select database();


# 创建一张表
create table stuinfo(
id int,
name varchar(20));

show tables;
+----------------+
| Tables_in_test |
+----------------+
| stuinfo        |
+----------------+
1 row in set (0.00 sec)


# 查看表结构
desc stuinfo;

# 查看表内的数据
select * from stuinfo;

# 查看数据库版本
select version();
```



### mysql 语法规范

- 不区分大小写，建议关键字大写，表名、列名小写
- 每条命令最好用分号结尾
- 每条命令根据需要，可以进行缩进或者换行
- 注释
  - 单行注释：# 注释文字
  - 单行注释： -- 注释文字
  - 多行注释： /* 注释文字 */



### DQL 语言的学习

1、基础查询

语法： select   查询列表   from 表名;

- 查询表中的单个字段

```
select name from test;
```

- 查询表中的多个字段

```
select name,id from test 
```

- 查询表中的所有字段

```
 select * from test
```

- 查询表达式

```
select 100*20;
select 100%98;
```

- 查询函数

```
select version();
```

- 起别名

> 便于理解，如果查询的字段有重名的情况，使用别名可以区分开来

方式一：使用 as

```
select 100%98 AS 结果;
select 100%98 AS 结果, 99+1 AS 和;
```

方式二：使用空格

```
select 100%98  结果;
select 100%98  结果, 99+1 AS 和;
select mingzi  AS "asdasdasd " from name
```

- 去重 DISTINCT

查询员工表中涉及到所有的部门编号

```
select  DISTINCT name  from t_user
```




- +号的作用

  运算符

  select 100+90;

案例：查询员工名和姓链接成一个字段，并显示未姓名

```
select  concat('a','b','c')  AS 结果;

select concat(`first_name`,',',`last_name`,',',`job_id`,',',IFNULL(commission_pct,0)) AS out_put from employees;  #concat拼接，当commission为null，则输出0
```