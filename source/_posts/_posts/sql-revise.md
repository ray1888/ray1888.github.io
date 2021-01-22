---
title: SQL复习
date: 2019-09-10 10:05:38
tags: SQL
---

<!-- ---
layout: post
title: SQL复习
author: Ray Chan(ray1888)
date: '2019-09-10 10:05:38 +0800'
category: sql
summary: sql-revise for interview 
thumbnail: sql.jfif
--- -->

# 目录
1. [Join](#Join)    
    1.1 [基本Join类型](#BasicJoin)  
    1.2 [高级Join类型](#AdvanceJoin)
2. [视图和子查询](#ViewNSubquery)  
    2.1 [视图](#View)  
    2.2 [子查询](#Subquery)  
3. [基本语法](#Gramma)  
    3.1 [Select](#Select)  
    3.2 [OrderBy](#OrderBy)   
    3.3 [Update](#Update)  
    3.4 [Delete](#Delete)  
4. [Having & GroupBy](#HavingNGroupBy)  
5. [注意事项](#Notice)  
6. [ShareNote](#ShareNote)  

# 目的
因为之前工作上面一直使用Python的ORM，然后很少使用手写SQL，因此需要回顾一些Sql的用法。

# <a id="Join"><span class="toptitle">Join</span></a> 
## 此处所提及的表的数据和表类型
```
TABLE_A
  PK Value
---- ----------
   1 FOX
   2 COP
   3 TAXI
   6 WASHINGTON
   7 DELL
   5 ARIZONA
   4 LINCOLN
  10 LUCENT

TABLE_B
  PK Value
---- ----------
   1 TROT
   2 CAR
   3 CAB
   6 MONUMENT
   7 PC
   8 MICROSOFT
   9 APPLE
  11 SCOTCH
```

## <a id="BasicJoin"><span class="secondtitle">基本Join类型</span></a>

### InnerJoin

<!-- ![InnerJoin](/assets/img/posts/sqlJoin/INNER_JOIN.png) -->
{% asset_img INNER_JOIN.png InnerJoin %}

求表A与表B数据的交集部分    

```
-- INNER JOIN
SELECT A.PK AS A_PK, A.Value AS A_Value,
       B.Value AS B_Value, B.PK AS B_PK
FROM Table_A A
INNER JOIN Table_B B
ON A.PK = B.PK

A_PK A_Value    B_Value    B_PK
---- ---------- ---------- ----
   1 FOX        TROT          1
   2 COP        CAR           2
   3 TAXI       CAB           3
   6 WASHINGTON MONUMENT      6
   7 DELL       PC            7

(5 row(s) affected)
```

### LeftJoin

<!-- ![InnerJoin](/assets/img/posts/sqlJoin/LEFT_JOIN.png) -->
{% asset_img LEFT_JOIN.png LeftJoin %}

选取表A的所有数据, 在此例子中，如果表B不存在对应查询的ID的时候，则会填入Null  

```
-- LEFT JOIN
SELECT A.PK AS A_PK, A.Value AS A_Value,
B.Value AS B_Value, B.PK AS B_PK
FROM Table_A A
LEFT JOIN Table_B B
ON A.PK = B.PK

A_PK A_Value    B_Value    B_PK
---- ---------- ---------- ----
   1 FOX        TROT          1
   2 COP        CAR           2
   3 TAXI       CAB           3
   4 LINCOLN    NULL       NULL
   5 ARIZONA    NULL       NULL
   6 WASHINGTON MONUMENT      6
   7 DELL       PC            7
  10 LUCENT     NULL       NULL

(8 row(s) affected)
```

### RightJoin

<!-- ![InnerJoin](/assets/img/posts/sqlJoin/RIGHT_JOIN.png) -->
{% asset_img RIGHT_JOIN.png RightJoin %}

选取表B的所有数据，在此例子中，如果表A不存在对应查询的ID的时候，则会填入Null  

```
-- RIGHT JOIN
SELECT A.PK AS A_PK, A.Value AS A_Value,
B.Value AS B_Value, B.PK AS B_PK
FROM Table_A A
RIGHT JOIN Table_B B
ON A.PK = B.PK

A_PK A_Value    B_Value    B_PK
---- ---------- ---------- ----
   1 FOX        TROT          1
   2 COP        CAR           2
   3 TAXI       CAB           3
   6 WASHINGTON MONUMENT      6
   7 DELL       PC            7
NULL NULL       MICROSOFT     8
NULL NULL       APPLE         9
NULL NULL       SCOTCH       11

(8 row(s) affected)
```

### OuterJoin

<!-- ![InnerJoin](/assets/img/posts/sqlJoin/FULL_OUTER_JOIN.png) -->
{% asset_img FULL_OUTER_JOIN.png FullOuterJoin %}

选取表A与表B的数据的全集,当另一张表缺失的情况下，会填补NUll信息  

```
-- OUTER JOIN
SELECT A.PK AS A_PK, A.Value AS A_Value,
B.Value AS B_Value, B.PK AS B_PK
FROM Table_A A
FULL OUTER JOIN Table_B B
ON A.PK = B.PK

A_PK A_Value    B_Value    B_PK
---- ---------- ---------- ----
   1 FOX        TROT          1
   2 COP        CAR           2
   3 TAXI       CAB           3
   6 WASHINGTON MONUMENT      6
   7 DELL       PC            7
NULL NULL       MICROSOFT     8
NULL NULL       APPLE         9
NULL NULL       SCOTCH       11
   5 ARIZONA    NULL       NULL
   4 LINCOLN    NULL       NULL
  10 LUCENT     NULL       NULL

(11 row(s) affected)
```

## <a id="AdvanceJoin"><span class="secondtitle">高级Join类型</span></a>

### LEFT JOIN EXCLUDING INNER JOIN

<!-- ![LeftJoin](/assets/img/posts/sqlJoin/LEFT_EXCLUDING_JOIN.png) -->
{% asset_img LEFT_EXCLUDING_JOIN.png LeftExcludeJoin %}

选择A与B中，A没有与B有交集的部分  
```
-- LEFT EXCLUDING JOIN
SELECT A.PK AS A_PK, A.Value AS A_Value,
B.Value AS B_Value, B.PK AS B_PK
FROM Table_A A
LEFT JOIN Table_B B
ON A.PK = B.PK
WHERE B.PK IS NULL

A_PK A_Value    B_Value    B_PK
---- ---------- ---------- ----
   4 LINCOLN    NULL       NULL
   5 ARIZONA    NULL       NULL
  10 LUCENT     NULL       NULL
(3 row(s) affected)
```

### RIGHT JOIN EXCLUDING INNER JOIN

<!-- ![InnerJoin](/assets/img/posts/sqlJoin/RIGHT_EXCLUDING_JOIN.png) -->
{% asset_img RIGHT_EXCLUDING_JOIN.png RightExcludeJoin %}


选择A与B中，B没有与A有交集的部分  
```
-- RIGHT EXCLUDING JOIN
SELECT A.PK AS A_PK, A.Value AS A_Value,
B.Value AS B_Value, B.PK AS B_PK
FROM Table_A A
RIGHT JOIN Table_B B
ON A.PK = B.PK
WHERE A.PK IS NULL

A_PK A_Value    B_Value    B_PK
---- ---------- ---------- ----
NULL NULL       MICROSOFT     8
NULL NULL       APPLE         9
NULL NULL       SCOTCH       11

(3 row(s) affected)
```

### OUTER JOIN EXCLUDING INNER JOIN

<!-- ![InnerJoin](/assets/img/posts/sqlJoin/OUTER_EXCLUDING_JOIN.png) -->
{% asset_img OUTER_EXCLUDING_JOIN.png InnerJoin %}

选择A与B中，A没有与B有交集的部分和B与A没有交集的部分  
```
-- OUTER EXCLUDING JOIN
SELECT A.PK AS A_PK, A.Value AS A_Value,
B.Value AS B_Value, B.PK AS B_PK
FROM Table_A A
FULL OUTER JOIN Table_B B
ON A.PK = B.PK
WHERE A.PK IS NULL
OR B.PK IS NULL

A_PK A_Value    B_Value    B_PK
---- ---------- ---------- ----
NULL NULL       MICROSOFT     8
NULL NULL       APPLE         9
NULL NULL       SCOTCH       11
   5 ARIZONA    NULL       NULL
   4 LINCOLN    NULL       NULL
  10 LUCENT     NULL       NULL

(6 row(s) affected)
```


# <a id="ViewNSubquery"><span class="toptitle">视图与子查询</span></a>

## <a id="View"><span class="secondtitle">视图</span></a>
视图并不是用来保存数据的，而是通过保存读取数据的SELECT 语句的方法来为用户提供便利的工具。  

究竟视图是什么呢？如果用一句话概述的话，就是“从SQL 的角度来看视图就是一张表”。实际上，在SQL 语句中并不需要区分哪些是表，哪些是视图。  
视图和表的差别：区别只有一个，那就是“是否保存了实际的数据”。  
但是使用视图时并不会将数据保存到存储设备之中，而且也不会将数据保存到其他任何地方。实际上视图保存的是SELECT 语句（图5-1）。我们从视图中读取数据时，视图会在内部执行该SELECT 语句并创建出一张临时表。

视图是需要通过create view 的语法来创建的，本质也是满足一定条件的数据的集合。


### 限制
1. 视图不可以与Group By 同时使用（可能某些DB会不支持）


## <a id="Subquery"><span class="secondtitle">子查询</span></a>

基础定义：子查询就是一次性的视图（SELECT语句）。与视图不同，子查询在SELECT语句执行完毕之后就会消失。  

```
select col_1 from (select col2_ from j  where $cond 1) p where $cond2

其中
p为子查询，内容为select col2_ from j  where $cond 1 满足这个条件的 j 表数据创建出来的新临时表
```

### 标量子查询

而标量子查询则有一个特殊的限制，那就是必须而且只能返回1 行1列的结果。也就是返回表中某一行的某一列的值。  
由于返回的是单一的值，因此标量子查询的返回值可以用在= 或者<> 这样需要单一值的比较运算符之中。  

#### 标量子查询的书写位置
可以再任意位置： SELECT 子句、GROUP BY 子句、HAVING 子句，还是ORDERBY 子句，几乎所有的地方都可以使用。  


# <a id="Gramma"><span class="toptitle">基本语法</span></a>

## <a id="Select"><span class="secondtitle">select</span></a>

基础的选择操作
```
select col1, col2, col..... from table1... where $cond
```

### Select 中 引用别名列
```
select * from (select sal as salary, comm as commission from emp) x where salary < 5000
```
上面的这种方法可以处理类似的情况：  
1. 聚合函数(Sum()、Min()、Max())
2. 标量子查询
3. 窗口函数
4. 别名
  
将含有别名列的查询放入内嵌视图，就可以在外层查询中引用别名列。为什么要这么做
呢？ WHERE 子句会比SELECT 子句先执行，就最初那个失败的查询例子而言，当WHERE 子句  
被执行时，SALARY 和COMMISSION 尚不存在。直到WHERE 子句执行完毕，那些别名列才会生  
效。然而，FROM 子句会先于WHERE 子句执行。如果把最初的那个查询放入一个FROM 子句，  
其查询结果会在最外层的WHERE 子句开始之前产生，这样一来，最外层的WHERE 子句就能  
“看见”别名列了。当表里的某些列没有被恰当命名的时候，这个技巧尤其有用。  
  
### 使用条件逻辑
```
select col1 , col2 ,
       case when col2 <= 2000 then "UnderPAID"
            when col2 > 4000 then "OVERPAID"
        end as status
    from emp 
```

输出的列为[col1, status]  

### 限制返回条数

```
 select * from emp limit 1
```

随机返回固定条数
```
select * from emp order by rand() limit 5 
```

### Null的判断

```
//  判断col1 是否为空
select * from emp where col1 is null
//  判断col2 是否不为空
select * from emp where col2 is not null
```

### Union 

```
select deptno from emp 
union 
select deptno from dept 
``` 
  
如果Union想获取到重复的条目，则应该使用union all   
```
// 如果使用union all 的等价实现
select distinct deptno from
(select deptno from emp 
union all 
select deptno from dept )
```

### 对于多表Join查询  

#### 合并相关的行
目的：对一个共同的列或者具有相同值的列做连接查询，返回多个表中的行。
```
select e.name, d.loc from emp e , dept d 
where e.deptno = 10 and e.deptno = d.deptno
```

这个解决方案是一个关于连接查询的例子。更准确地说，它是内连接中的相等连接。连接
查询是一种把来自两个表的行合并起来的操作。对于相等连接而言，其连接条件依赖于某
个相等条件（例如，一个表的部门编号和另一个表的部门编号相等）。内连接是最早的一
种连接，它返回的每一行都包含了来自参与连接查询的各个表的数据。

理论上，连接操作首先会依据FROM 子句里列出的表生成笛卡儿积（列出所有可能的行组
合），如下所示。

<!-- ![Union1](/assets/img/posts/sqlPost/Union-1.png) -->
{% asset_img Union-1.png Union1 %}
EMP 表里部门编号为10 的全部员工与DEPT 表的所有部门组合都被列出来了。然后，通过
WHERE 子句里的e.deptno 和d.deptno 做连接操作，限定了只有EMP.DEPTNO 和DEPT.DEPTNO
相等的行才会被返回。
<!-- ![Union2](/assets/img/posts/sqlPost/Union-2.png) -->
{% asset_img Union-2.png Union2 %}

```
类似实现，使用显式的Join来实现
select e.name, d.loc from emp e  
   inner join  dept d on e.deptno = d.deptno
   where e.deptno = 10 
```

如果把上面的变成小于等于10的情况,用Join的方式合并的话会变成这样
```
select e.name, d.loc from emp e  
   inner join  dept d on e.deptno = d.deptno
   where (e.deptno = 10) or (e.deptno < 10)
```

#### 查找两个表相同的行并且连接多列

需求： 获取Clerk的信息，但是需要全部的列  

步骤： 1. 创建一个视图把Clerk的信息查出来  
       2. 然后再去与emp表做Join获取完整的信息  
```
select e.empno, e.name, e.job, e.sal, e.deptno from emp e, 
(select ename, job, sal from emp where job = "CLERK") V 
where V.ename = e.ename and 
      V.job = e.job and 
      V.sal = e.sal
```

Join的处理手法  
```
select e.empno, e.name, e.job, e.sal, e.deptno from emp e
join (select ename, job, sal from emp where job = "CLERK") V on (
    V.ename = e.ename
    V.job = e.job
    V.sal = e.sal)
```


#### 查询只存在于一个表中的数据

一般来说，直接使用not in 就可以了。但是对于如果含有Null的数据，就不能直接使用这样的方法处理。  
那为什么null的数据就会出现问题呢？这个就要看一下他可能的实现方式  
对于Mysql的实现， not in 和 in  本质上是  or的关系运算。 由于null 参与Or的逻辑运算方式不一致，In 和Not in 将产生不同的结果。  

此处默认表中有一条Null的数据
```
// In
select deptno from dept where deptno in (10, 50, null)
~~~~~~~~~~
Deptno
--------
10

select deptno from dept where (deptno=10 or deptno=50 or deptno=null)
~~~~~~~~~~
Deptno
--------
10

//Not in 

select deptno from dept where deptno not in (10, 50, null)
~~~~~~~~~~
Deptno
--------
no rows

select deptno from dept where deptno not  (deptno=10 or deptno=50 or deptno=null)
~~~~~~~~~~
Deptno
--------
no rows

```

如果想要解决上面的null 所导致的问题， 需要结合Not exists 和关联子查询。 
```
select d.deptno from dept d where
not exists (select null from emp e where d.deptno = e.deptno)
~~~~~~~~~
Deptno
--------
40
```


#### 确定两个表是否有相同的数据

<!-- ![Union3](/assets/img/posts/sqlPost/Union-3.png) -->
对于上面这种查询，可能会出现红色圈的数据重复的现象。那么我们要怎样才能确定是否有重复数据呢？  
{% asset_img Union-3.png Union3 %}

```
create view v as 
select * from emp where deptno != 10
union all 
select * from emp where ename = "ward"

```

处理手法使用关联子查询和UNION ALL 找出那些存在于视图V 而不存在于EMP 表的数据，以及存在于EMP 表而不存在于视图V 的数据，并将它们合并起来。  

```
//  处理存在于EMP 不存于v的查询
select * from (
   select e.empno, e.ename, e.job, e.mgr, e.hiredate,
          e.sal, e.comm, e.deptno, count(*) as cnt from emp e
          group by empno, ename, job, mgr, hiredate,
                   sal, comm, deptno) e 
) where not exists (
   select null from (
      select v.empno, v.ename, v.job, v.mgr, v.hiredate,
          v.sal, v.comm, v.deptno, count(*) as cnt from v
          group by empno, ename, job, mgr, hiredate,
                   sal, comm, deptno) v
      where v.empno = e.empno and 
            v.ename = e.ename and
            v.job = e.job and
            v.mgr = e.mgr and
            v.hiredate = e.hiredate and
            v.sal = e.sal and
            v.deptno = e.deptno and
            v.cnt = e.cnt and 
            coalesce(v.comm, 0) = coalesce(e.comm, 0)
   )  
)
```

```
//  处理存在于V 不存于EMP的查询
select * from (
   select v.empno, v.ename, v.job, v.mgr, v.hiredate,
          v.sal, v.comm, v.deptno, count(*) as cnt from v
          group by empno, ename, job, mgr, hiredate,
                   sal, comm, deptno) v
) where not exists (
   select null from (
      select e.empno, e.ename, e.job, e.mgr, e.hiredate,
          e.sal, e.comm, e.deptno, count(*) as cnt from v
          group by empno, ename, job, mgr, hiredate,
                   sal, comm, deptno) e
      where v.empno = e.empno and 
            v.ename = e.ename and
            v.job = e.job and
            v.mgr = e.mgr and
            v.hiredate = e.hiredate and
            v.sal = e.sal and
            v.deptno = e.deptno and
            v.cnt = e.cnt and 
            coalesce(v.comm, 0) = coalesce(e.comm, 0)
   )  
)
```

```
// 总体
select * from (
   select e.empno, e.ename, e.job, e.mgr, e.hiredate,
          e.sal, e.comm, e.deptno, count(*) as cnt from emp e
          group by empno, ename, job, mgr, hiredate,
                   sal, comm, deptno) e 
) where not exists (
   select null from (
      select v.empno, v.ename, v.job, v.mgr, v.hiredate,
          v.sal, v.comm, v.deptno, count(*) as cnt from v
          group by empno, ename, job, mgr, hiredate,
                   sal, comm, deptno) v
      where v.empno = e.empno and 
            v.ename = e.ename and
            v.job = e.job and
            v.mgr = e.mgr and
            v.hiredate = e.hiredate and
            v.sal = e.sal and
            v.deptno = e.deptno and
            v.cnt = e.cnt and 
            coalesce(v.comm, 0) = coalesce(e.comm, 0)
   )  
)
Unoin all
select * from (
   select v.empno, v.ename, v.job, v.mgr, v.hiredate,
          v.sal, v.comm, v.deptno, count(*) as cnt from v
          group by empno, ename, job, mgr, hiredate,
                   sal, comm, deptno) v
) where not exists (
   select null from (
      select e.empno, e.ename, e.job, e.mgr, e.hiredate,
          e.sal, e.comm, e.deptno, count(*) as cnt from v
          group by empno, ename, job, mgr, hiredate,
                   sal, comm, deptno) e
      where v.empno = e.empno and 
            v.ename = e.ename and
            v.job = e.job and
            v.mgr = e.mgr and
            v.hiredate = e.hiredate and
            v.sal = e.sal and
            v.deptno = e.deptno and
            v.cnt = e.cnt and 
            coalesce(v.comm, 0) = coalesce(e.comm, 0)
   )  
)
```

### <a id="AdvanceApp"><span class="thirdtitle">高级应用</span></a>
#### 分页

可以使用Mysql 或者PG 内置的 limit 和 offset 进行处理 
```
select sal from emp order by sal limit 5 offset 0

select sal from emp order by sal limit 5 offset 5 
```

#### 间隔获取记录
对于没有默认编号的数据，我们需要先编号再进行过滤的操作
如果默认是已经有字段是序号的话，则采用直接Mod数据即可
```
select x.name from (
   select a.ename , (
      select count(*) from emp b where
      b.ename <= a.ename
   ) as rn from emp a 
) x where mod(rn,2) = 1
```

#### 外查询使用OR逻辑
1. 先去Join表，然后再去进行Or的逻辑判断
```
select e.ename, d.deptno , d.dname, d.loc from dept d 
left join emp e on (d.deptno = e.deptno
                    and (e.deptno=10 or e.deptno=20))
order by 2 
```

2. 先创建一个中间表，然后再去进行Join的操作

select e.ename, d.deptno , d.dname, d.loc from dept d
left join (select * from emp e where e.deptno=10 or e.deptno=20)
on d.deptno = e.deptno order by 2 


#### 对单表需要做数据运算情况
情况1： 找出互逆的记录（本例）  
情况2： 查找表中某列1相差为1，并且某列2差为5的记录  

总体的思路，把自己与自己(或者与自己的子集)求笛卡尔积，然后去进行条件的筛选

```
select distinct v1.* from V v1, V v2 where
        v1.test1 = v2.test2
        and v1.test2 = v2.test1
        and v1.test1 <= v1.test2
```

#### 找出最靠前的N条记录

此处使用了标量子查询来创建了一张临时表的RNK的列
```
// 
select ename, sal from (
   select (
      select (count(distinct b.sal) from emp b where 
             a.sal <= b.sal) as rnk,
             a.sal,
             a.ename
   ) from emp a 
) where rnk <=5
```



## <a id="OrderBy"><span class="secondtitle">OrderBy</span></a>

### 基础查询
```
// 升序查询
select * from emp order by col2 asc;
// 降序查询
select * from emp order by col2 desc;
```

### 多字段排序
```
select empno, deptno, sal, ename, job from emp order by deptno (asc), sal desc;
```

### 动态排序
```
select ename, sal, job, comm from emp order by 
       case when job = "salesman" then comm
            else  sal 
       end;
```


## <a id="Update"><span class="secondtitle"> update </span></a>

### 基础语法
```
update table name set col_name = xxx where $cond
```

## <a id="Delete"><span class="secondtitle">delete</span></a>

基础语法  
```
delete from table_name where $cond
```

删除重复记录
```
delete from table where id not in (select min(id) from table group by name)
```

# <a id="HavingNGroupBy"><span class="toptitle">Having & GroupBy</span></a>
```
wiki原文
A HAVING clause in SQL specifies that an SQL SELECT statement should only return rows where aggregate values meet the specified conditions. It was added to the SQL language because the WHERE keyword could not be used with aggregate functions.
The HAVING clause filters the data on the group row but not on the individual row.
To view the present condition formed by the GROUP BY clause, the HAVING clause is used.
```

Having的语句是必须要在GroupBy后面才能使用。并且与Where的区别是，Where不能直接接入聚合的函数（如Sum()、Count()、Avg()） 这种的聚合函数， 意思是不能 where sum(column_a) 这样的用法）, 并且Having可以对按Group区分的Row进行过滤的操作

所以常规语法一般是
```
select * from table_a A group by columa_a having count (A.column_a ) > 200
```

# <a id="Notice"><span class="toptitle">特殊注意</span></a>

1. 类似于Sum， max， min , avg 这些也是可以直接用于select 的条件上面的
```
select max(Salary) as SecondHighestSalary from employee where salary<(select max(distinct(salary)) from employee)
```

2. sql 三元运算符
if (expr1, expr2, expr3)
跟正常编程语言中的三元运算符一致，只是语法有变动。也是满足条件一，则返回expr2，否则返回expr3



# <a id="ShareNote"><span class="toptitle">ShareNote</span></a>
1. [Visual-Representation-of-SQL](https://www.codeproject.com/Articles/33052/Visual-Representation-of-SQL-Joins)
2. [Having-Sql-Cluse-wiki](https://en.wikipedia.org/wiki/Having_(SQL))
3. [Sql经典实例](https://book.douban.com/subject/30259463/)
4. [Sql基础教程](https://www.ituring.com.cn/book/miniarticle/47448)