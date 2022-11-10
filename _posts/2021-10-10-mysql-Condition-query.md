---
layout: post
title: mysql 条件查询
date: 2021-10-10 
tags: Mysql
---

### 条件查询

```
select 
	查询列表
from
	表名
where
	筛选条件;
```

### 分类

<b>1. 按条件表达式筛选</b>

条件运算符: ` >  <  =   !=  <>  >=  <=`

<b>2.  按逻辑表达式筛选</b>

逻辑运算符：` &&  ||  !   and  or  not `

<b>3. 模糊查询</b>

`Like` 

`between and ` 

`in `

`is null`



### 一、按条件表达式筛选

```sql
select * from employees where salary>12000;   #查询工资大于12000的
```

```sql
select last_name,department_id from employees where department_id<>90; # 查询部门编号不等于 90 号的员工名和部门编号
```



### 二、按逻辑表达式筛选

案例1：查询工资在10000到20000之间的员工名、工资以及奖金

```sql
select last_name,salary,commission_pct from employees where salary>=10000 AND salary<=20000;
```

案例2：查询部门编号不是在90到110之间，或者工资高于15000的员工信息

```sql
select * from employees where department_id<90 or department_id >110 or salary>15000
```

### 三、模糊查询

- like
- between and
- in 
- is null 和 is not null



#### like

> 一般和通配符搭配使用，
>
> %代表多个字符，包含0个字符
>
> _ 任意单个字符

案例1：查询员工名中包含字符a的员工信息

```
select * from employees where last_name like '%a%';  # %代表通配符
```

案例2：查询员工名中第三个字符为e，第五个字符为a的员工名和工资

```sql
select  last_name,salary from employees where last_name like '__e_a%';
```

案例3：查询员工名中第二个字符为_的员工名

```sql
select last_name from employees where last_name like '_\_%';
```

#### between and

> 可以提高语句的简洁度，包含临界值，两个临界值不要调换顺序

案例1： 查询员工编号在100到120之间的员工信息

```
select * from employees where employee_id >=100 AND employee_id <=120;

select * from employees where employee_id between 100 AND 120;
```

#### in

> 用于去判断某字段的值是否属于 in 列表中的某一项
>
> 使用 in 提高语句简洁度
>
> In 列表的值类型必须一致或者兼容

案例：查询员工的工种编号是 IT_PROG、AD_VP、AD_PRES中的一个员工名和工种编号

```
select last_name,job_id from  employees where job_id = 'IT_PROG' or job_id = 'AD_VP' or job_id = 'AD_PRES';

select ast_name,job_id from  employees where job_id IN('ITPROG','AD_VP','AD_PRES');
```

#### is null

> = 或者 <>不能用于判断 null值
>
> is null 或者 is not null 可以

案例1：查询没有奖金的员工名和奖金率

```
select last_name,commission_pct from employees where commission_pct IS NULL;
```

案例2：查询有奖金的员工名和奖金率

```sql
select last_name,commission_pct from employees where commission_pct IS NOT NULL;
```

#### 案例

查询没有奖金且工资小于18000的salary,last_name

```sql
select salary,last_name from employees where commision_pct is null and salary < 18000;
```

