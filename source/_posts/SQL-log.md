---
title: SQL.log
date: 2018-08-23 14:36:53
tags: [SQL, MYSQL]
categories: 数据库
respository: git@github.com:Gfeather/feather.git
branch: master
---
*learning ,  perception design* 

*对SQL基础知识的简单汇总*



# SQL_LOG
[SQL语句种类](#SQL语句种类)
[查询](#查询)
[数据更新](#数据更新)
[复杂查询](#复杂查询)
> 视图
> 子查询
> 关联子查询

[函数、谓词和CASE表达式](#函数、谓词和CASE表达式)
> 函数
> 谓词
> CASE表达式

[集合运算](#集合运算)
> 表的加减法
> 联结

[SQL高级处理](#SQL高级处理)
> 窗口函数
> GROUPING运算符


## SQL语句种类
* [DDL(Data Definition Language)](#DDL) 数据定义语言
> CREATE
> DROP
> ALTER 
* [DML(Data Manipulation Language)](#DML) 数据操纵语言
> SELECT
> INSERT
> DELETE
> UPDATE
* [DCL(Data control Language)](#DCL) 数据控制语言
> COMMIT
> ROLLBACK
> GRANT	赋予用户操作权限
> REVOKE 取消用户操作权限


#### SQL书写规则
* 以`;`结尾
* 不区分大小写
		
		SHOW DATABASES;//展示所有数据库
		SHOW TABLES;//展示当前数据库中所有表
		DESCRIBE <table>;//展示表结构
		DESC <table>;
		truncate table;清空表、包括约束；

#### DDL语句<p id='DDL'/>

* CREATE
```
CREATE DATAVASE <database>;//创建数据库
//创建表
CREATE TABLE <table>(
	<column1> <type> <constraint>,
	<column2> <type> <constraint>,
			   ...
	<TABLE constraint>;
);
```
* DROP
```
//be careful
DROP <database>;
DROP <table>;
```
* ALTER
```
ALTER TABLE <table> ADD COLUMN <column> [first|before|after][<column>];
ALTER TABLE <table> DROP COLUMN <column>;
ALTER TABLE <table> ALTER COLUMN <column>; 
ALTER TABLE <table> CHANGE <oldcolumn> <newcolumn>;
ALTER TABLE <table> MOTIFY <column>;  
```

#### DML语句<p id='DML'/>

* INSERT
```
INSERT INTO <table> VALUES (value1, value2...);
INSERT INTO <table> (column1, column2...) VALUES (value1, value2...);
```
* DELETE
```
DELETE FROM <table> WHERE ...;
DELETE FROM <table>;
```
* UPDATE
```
UPDATE <table> SET COLUMN = new WHERE ...;
```
* SELECT
```
//base
SELECT <column> FROM <table>;
```

#### DCL语句<p id='DCL'/>

## 查询<p id='查询'/>

查询语句结构：
```
SELECT <column> 
FROM <table>
[WHERE <condition>]
[GROUP BY <condition>]
[HAVING <condition>]
[ORDER BY <column> <order>];
```
各子句功能：
- SELECT， 在使用GROUP BY时，应只包含聚合键、聚合函数，来展示聚合结果
- WHERE 子句，支持比较运算符、逻辑运算符，**对每一行进行数据操作**，所以不能使用聚合函数
- GROUP BY 子句，根据聚合键对行进行分组
- HAVING 子句，只能使用聚合键、聚合函数，使用组特征对组进行选择
- ORDER BY 子句，根据排序键进行排序，不使用ORDER BY前排列为随机顺序，`ASC↑ DESC↓`
- 执行顺序：

```
FROM -> WHERE -> GROUP BY -> HAVING -> ORDER BY -> SELECT
```


***null***： 
* 在算数运算中，对null的算术运算结果都为null，eg：1 + null = null
* 在比较运算中，null不能使用比较运算符，eg： `where <column> <> null;` 结果为空
* 在逻辑运算中，可以使用谓词判断是否为空，eg： `IS null` / `IS NOT null`
* 三值逻辑，因为null，SQL中的逻辑运算被称为三值逻辑，即存在真、不确定、假，三种状态，真假程度为此顺序。
* 在聚合中，使用聚合函数会排除含null值的行(*会选中全部行)。
* 在分组中，聚合键中含null时，会将值null的行作为一组；
* 在排序中，排序键中含null的行则会被放到开头或结尾，排序为随机


## 数据更新<p id='数据更新'/>

* INSERT

> 多行插入，`INSERT INTO <table> VALUES (value1...),VALUES (value1...)...;`
> 插入默认值，显式： `DEFAULT` 显式代替默认值，隐式：INSERT时在字段列表和values列表省略该值。
> 从表中插入，使用`INSERT...SELECT`语句，将SELECT查询出的数据插入，支持完整SELECT中的选择、聚合、排序等操作用于表复制；

* DELETE
```
DELETE FROM <table>
WHERE <column>
```

* UPDATE
```
UPDATE <table>
SET <column>
WHERE <column>
```
> 多列更新，SET 指令后允许对多个列进行更新

## 复杂查询<p id='复杂查询'/>
* 视图
用于保存复杂查询，在SQL中视图作为一个表，但是不存储数据，本体为一个SELECT查询。
	- 创建视图不能使用ORDER BY，因为视图作为一个表是无序的
	- 视图更新的条件：没有使用DISTINCT、FROM子句中只有一张表、没有使用GROUP BY、没有使用HAVING，即保证视图中查询的数据都来自一张表、数据都存在且没有做聚合。
语法：

```
CREATE VIEW (<column>...)
AS
SELECT....;
```
	

* 子查询
简单的讲是一个一次性的视图，即在一个SELECT查询中查询数据。

```
SELECT <column>
FROM 
  (SELECT <column>
  FROM <table>);
```

常量子查询：
查询结果是一个值，可以当做一个数值来为列赋值
* 关联子查询
WHERE 子句是对行的操作，下面的查询用一个键将子查询和主查询关联，来取出关联键的分组的聚合值


	SELECT <column> 
	FROM <table> AS p1
	where <column3> > ( SELECT AVG(<column3>)
					FROM <table> AS p2
					WHERE p1.<column2> = p2.<column2>
					GROUP BY <column2> );
//查询表中某列指标大于所属类别平均值的行


## 函数、谓词和CASE表达式<p id='函数、谓词和CASE表达式'/>

* 函数
	* 聚合函数
	* 算术函数
	* 字符串函数
	* 日期函数
	* 转换函数
* 谓词
> * LIKE 用于对字符串的模糊匹配，支持通配符
> LIKE 匹配分为前方一致、中间一致、后方一致，通配符`%`表示一个或多个字符、`_`表示单个字符
> * BETWEEN 范围查询 eg：`BETWEEN a AND b` 相当于  a <= x <= b
> * IS NULL/IS NOT NULL 判断是否为NULL
> * IN 判断是否在一组值、查询中
> * EXIST 判断集合是否存在元素，对象一般为一个子查询

## 集合运算<p id='集合运算'/>
* 表的加减法
作为运算对象的列数、数据类型都要相同

> * 并集(`UNION`)
 
> 
>     SELECT * FROM <table>
>     UNION [ALL]//包括重复项目，一行的所有数据都相等才为相同记录
>     SELECT * FROM <table>;


> * 交集(`INTERSECT`)
> 用法与`UNION`相同，mysql不支持，需要取所有记录通过筛选出现次数获得


>     SELECT * 
>     FROM (
>        SELECT * FROM <table>
>        UNION ALL
>        SELECT * FROM <table>) t
>     GROUP BY id
>     HAVING COUNT(id) = 2;

 
> * 差集(`EXCEPT`)
> 用法与`UNION`相同，mysql不支持，需要取所有记录通过筛选出现次数获得
> 
> 
>     SELECT * 
>     FROM (
>        SELECT * FROM <table>
>        UNION ALL
>        SELECT * FROM <table>) t
>     GROUP BY id
>     HAVING COUNT(id) = 1;
> 
> 
* 联结
> * 内联结(`INNER JION`)
> 通过联结键将两个表中都存在的记录联结在一起
 

>     SELECT *
>     FROM p1 INNER JOIN p2
>      ON p1.id = p2.id;


> * 外联结(`OUTER JOIN`)
> 分为LEFT、RIGHT，用来指定主表以主表联结键为主来从从表获取对应记录

## SQL高级处理<p id='SQL高级处理'/>
* 窗口函数(OLAP OneLine Analytical Processing)

```
//窗口函数分为1、聚合函数(COUNT、MAX、MIN、SUM、AVG)2、RANK、DENSE_RANK、ROW_NUMBER等专门的窗口函数
//窗口函数就是将记录切割为窗口，再计算排序
//窗口函数只能用于SELECT子句中，因为它是对查询出的记录进行分组、排序，所以结果是无序的
//PATITION BY用来分组，ORDER BY计算排序
<OLAP> OVER ([PATITION BY <column>]
			 ORDER BY <colum>
			[ROWS n PRECEDING/FOLLING]) 	
```

> RANK函数， 计算排序时，具有相同位次的记录会占有序号，eg:1，1，1，4
> DESE_RANK函数，计算排序时，具有相同位次的记录不占有序号，eg:1，1，1，2
> ROW_NUMBER函数，分配连续序号，eg:1，2，3，4

```
//在使用聚合函数时，会以当前记录为基准，只统计'在自己之上的记录'，参考窗口方向
//下面的SQL会计算累计的平均值，即当前记录之前所有记录的平均值
SELECT id, 
	AVG(price) OVER (ORDER BY price) AS price_AVG
FROM <table>
//移动平均，在计算聚合函数时，可以指定窗口大小，和窗口方向
SELECT id, 
	AVG(price) OVER (ORDER BY price ROWS n PRECEDING/FOLLOING) AS price_AVG
FROM <table>
```

* GROUPING运算符
用来同时计算合计和小计

> * ROLLUP，一次计算出不同聚合键组合的结果
 
  
>     SELECT product_type, SUM(sale_price)
>     FROM product
>     GROUP BY ROLLUP(product_type);
>     //mysql中
>     SELECT product_type, SUM(sale_price)
>     FROM product
>     GROUP BY product_type WITH ROLLUP;
>     //相当于
>     GROUP BY()
>     GROUP BY(product_type)


> 以下运算符mysql均不支持
> * GROUPING函数，用来判断是否为超级分组记录的null，是返回1，不是返回0。
> * CUBE，在ROLLUP基础上添加对每个聚合键的计算
> * GROUPING SETS从ROLLUP或CUBE中选取部分列




*摘自《SQL基础教程》*