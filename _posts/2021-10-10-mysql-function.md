---
layout: post
title: mysql 函数
date: 2021-10-10 
tags: Mysql
---
### 排序查询

> 1、asc 代表的是升序，desc代表的是降序，如果不写，默认是升序
>
> 2、order by 字句中可以支持单个字段、多个字段、表达式、函数、别名
>
> 3、order by一般是放在查询语句的最后面

语法：

```sql
select 查询列表 from 表 [where 筛选条件] order by  排序列表 [asc|desc];
```

案例：查询员工，要求工资和从高到底排序

```sql 
select  * from employees order by salary desc;
select  * from employees order by salary asc;
```

案例2：查询部门编号>=90的员工信息，按入职时间的先后进行排序[添加筛选条件]

```sql
select * from employees wheres department_id>=90 order by hiredate asc;
```

案例3：按年薪的高低显示员工的信息和年薪[按表达式排序]

```sql
select *,salary*12*(1+IFNULL(commission_pct,0)) 年薪 from employees order by salary*12*(1+IFNULL(commission_pct,0)) desc;
```

案例4：按年薪的高低显示员工的信息和年薪[按别名排序]

```sql
select *,salary*12*(1+IFNULL(commission_pct,0)) 年薪 from employees order by 年薪 desc;
```

案例5：按姓名的长度显示员工的姓名和工资[按函数排序]

```sql
select length(last_name) 字节长度,last_name,salary from employees order by length(last_name) desc;
```

案例6：查询员工信息，要求先按工资排序，再按员工编号排序[按多个字段排序]

```sql
select * from employees order by salary asc,employee_id desc; 
```

