---
layout: post
title: mysql 约束
date: 2021-10-10 
tags: Mysql
---

### 常见约束

> 含义：一种限制，用于限制表中的数据，为了保证表中的数据的准确和可靠性

分类：六大约束

- NOT NULL：非空，用于保证该字段的值不能为空，比如姓名、学号
- DEFAULT：默认，用于保证该字段有默认值
- PRIMARY KEY：主键，用于保证该字段的值具有唯一性，并且非空
- UNIQUE：唯一约束，保证该字段的值具有唯一性，可以为空
- CHECK：检查约束【mysql中不支持】
- FOREIGN KEY：外键，用于限制两个表的关系，用于保证该字段的值必须来自于主标的关联列的值
  - 在从表添加外键约束，用于引用主表中某列的值

<b>添加约束的时机：</b>

 ​	1.创建表时

 ​	2.修改表时



<b>约束的添加分类：</b>

 ​	1.列级约束：六大约束语法上都支持，但是外键约束没有效果

 ​	2.表级约束：除了非空、默认，其他的都支持

### 创建表时添加约束

##### 1.添加列级约束

> 语法：
>
> ​		直接在字段名和类型后面追加约束类型即可。只支持：默认、非空、主键、唯一

```sql
create database students;
use students;
create table stuinfo(
	id INT PRIMARY KEY, #主键
	stuName VARCHAR(10) NOT NULL, #非空
  gender CHAR(1) CHECK (gender='男' OR gender='女')， #检查
  seat INT UNIQUE, #唯一
  age INT DEFAULT 18, #默认
  majorID INT references major(id) #外键
);

create table major(
	id INT PRIMARY KEY,
  majorName VARCHAR(20)
); 
	

```

##### 2.添加表级约束

> 语法：在各个字段的最下面
>
> 【constraint  约束名】 约束类型(字段名)

```sql
create table stuinfo(
	id INT,
  stuname VARCHAR(20),
  gender CHAR(1),
  seat INT,
  age INT,
  majorid INT,
  
  CONSTRAINT pk PRIMARY KEY(id), #主键
  CONSTRAINT uq UNIQUE(seat), #唯一键
  CONSTRAINT ck CHECK(gender ='男' OR gender='女'), #检查
  CONSTRAINT fk_stuinfo_major FOREIGN KEY(majorid) REFERENCES major(id) #外键
);
```

通用的写法

```sql
create table if not exists stuinfo(
	id INT PRIMARY KEY,
  stuname VARCHAR(20) NOT NULL,
  sex CHAR(1),
  age INT DEFAULT 18,
  seat INT UNIQUE,
  majorid INT,
  CONSTRAINT fk_stuinfo_major FOREIGN KEY(majorid) REFERENCES major(id)
)
```

