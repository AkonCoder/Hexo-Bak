---
title: '基于《MSSQL2008技术内幕:T-SQL语言基础》sql总结'
date: 2017-11-02 00:17:58
tags: sql server
comments: true
categories: sql server
---
#### SQL Server的体系结构
**Sql Server ** 是我们工作开发当中必不可少的，掌握sql server基础，熟练运用sql，学会sql server的调优成为大多数公司聘用工程师的基本要求，很多公司内部也要求工程师基于现有业务，能够很好的排查慢查询，定位到数据库查询的问题所在，进行索引、查找的优化。为此掌握sql server基础，明白sql server查询的原理和运行机制，可以更加有效的帮助我们去解决问题。不管基于自身对sql server的学习，还是公司层面的要求，这本书对于提高自身sql server知识还是挺不错的。
<!-- more -->
##### sql server数据库的组成
sql server数据库存储部分，也就是物理组成部分，在物理上由数据文件和事务日志文件组成，每个数据库必须至少有一个数据文件和一个日志文件。可以分为：
*mdf*
*ldf*
*ndf*

![sql server组成](http://oy40mv0rj.bkt.clouddn.com/2017-11-02-sqlserver02.jpg)

>其中的.mdf代表Master Data File，.ldf代表Log Data File，而.ndf代表Not Master Data File（非主数据文件）

##### 架构（Schema）和对象

一个数据库包含多个架构，而每个架构又包括多个对象。可以将架构看作是各种对象的容器，这些对象可以是表（table）、视图（view）、存储过程（stored procedure）等等。
![schema](http://oy40mv0rj.bkt.clouddn.com/2017-11-02-sqlserver03.png)

此外，架构也是一个命名空间，用作对象名称的前缀。例如，架设在架构Sales中有一个Orders表，架构限定的对象名称是Sales.Orders。如果在引用对象时省略架构名称，SQL Server将采用一定的办法来分析出架构名称是什么。如果不显示指定架构，那么在解析对象名称时，就会要付出一些没有意义的额外代价。因此，建议都加上架构名称。

#### 查询

##### 单表查询
（1）单表查询很简单，但是很多人不注意的地方就是喜欢用select *
```
SELECT  * FROM Sales.Shippers
```
select * 在表字段较多的时候，会出现性能下降；即使表字段比较少，我们也需要指明查询的具体字段，这样方便别人可以通过sql语句，明白查找的内容。
按需所取自己需要用到的字段信息。
（2）关于FROM子句：显示指定架构名称
　　通过显示指定架构名称，可以保证得到的对象的确是你原来想要的，而且还不必付出任何额外的代价。
（3）关于TOP子句：T-SQL独有关键字
① 可以使用PERCENT关键字按百分比计算满足条件的行数
```
SELECT TOP 1 PERCENT orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC;
```
上面这条SQL就会请求最近更新过的前1%个订单。
② 可以使用WITH TIES选项请求返回所有具有相同结果的行
```
SELECT TOP 5 WITH TIES orderid, orderdate, custid, empid
FROM Sales.Orders
ORDER BY orderdate DESC;
```
上面这条SQL请求返回与TOP n行中最后一行的排序值相同的其他所有行。
（4）关于OVER子句：为行定义一个窗口以便进行特定的运算
OVER子句的优点在于能够在返回基本列的同时，在同一行对它们进行聚合；也可以在表达式中混合使用基本列和聚合值列。

>例如，下面的查询为OrderValues的每一行计算当前价格占总价格的百分比，以及当前价格占客户总价格的百分比 。

```
SELECT orderid, custid, val,
100.0 * val / SUM(val) OVER() AS pctall,
100.0 * val / SUM(val) OVER(PARTITION BY custid) AS pctcust
FROM Sales.OrderValues;
```
（5）子句的逻辑处理顺序
<center>
![执行顺序](http://oy40mv0rj.bkt.clouddn.com/2017-11-02-sqlserver-03.png)
</center>

（6）运算符的优先级
<center>
![运算优先级](http://oy40mv0rj.bkt.clouddn.com/2017-11-02-sqlserver-04.png)
</center>

（7）CASE表达式

① 简单表达式：将一个值与一组可能的取值进行比较，并返回满足第一个匹配的结果；
```
SELECT productid,productname,categoryid,categoryname=(
    CASE categoryid
        WHEN 1 THEN 'Beverages'
        WHEN 2 THEN 'Condiments'
        WHEN 3 THEN 'Confections'
        WHEN 4 THEN 'Dairy Products'
        ELSE 'Unkonw Category'
    END)
FROM Production.Products;
```

② 搜索表达式：将返回结果为TRUE的第一个WHEN逻辑表达式所关联的THEN子句中指定的值。如果没有任何WHEN表达式结果为TRUE，CASE表达式则返回ELSE子句中出现的值。（如果没有指定ELSE，则默认返回NULL）；

```
SELECT orderid, custid, val, valuecategory=(
  CASE
    WHEN val < 1000.00    THEN 'Less than 1000'
    WHEN val BETWEEN 1000.00 AND 3000.00 THEN 'Between 1000 and 3000'
    WHEN val > 3000.00    THEN 'More than 3000'
    ELSE 'Unknown'
  END
)
FROM Sales.OrderValues
```
（8）三值谓词逻辑：TRUE、FALSE与UNKNOWN

SQL支持使用NULL表示缺少的值，它使用的是三值谓词逻辑，代表计算结果可以使TRUE、FALSE与UNKNOWN。在SQL中，对于UNKNOWN和NULL的处理不一致，这就需要我们在编写每一条查询语句时应该明确地注意到正在使用的是三值谓词逻辑。

> 例如，我们要请求返回region列不等于WA的所有行，则需要在查询过滤条件中显式地增加一个队NULL值得测试：

```
SELECT custid, country, region, city
FROM Sales.Customers
WHERE region <> N'WA'
OR region IS NULL;
```
　　另外，T-SQL对于NULL值得处理是先输出NULL值再输出非NULL值得顺序，如果想要先输出非NULL值，则需要改变一下排序条件，例如下面的请求：
```
select custid, region
from sales.Customers
order by (case
when region is null then 1 else 0
end), region;
```
　　当region列为NULL时返回1，否则返回0。非NULL值得表达式返回值为0，因此，它们会排在NULL值（表达式返回1）的前面。如上所示的将CASE表达式作为第一个拍序列，并把region列指定为第二个拍序列。这样，非NULL值也可以正确地参与排序，是一个完整解决方案的查询。

（9）LIKE谓词的花式用法

① %（百分号）通配符
```
SELECT empid, lastname
FROM HR.Employees
WHERE lastname LIKE N'D%';
```
② _（下划线）通配符：下划线代表任意单个字符

下面请求返回lastname第二个字符为e的所有员工
```
SELECT empid, lastname
FROM HR.Employees
WHERE lastname LIKE N'_e%';
```
③ [<字符列>]通配符：必须匹配指定字符中的一个字符

　下面请求返回lastname以字符A、B、C开头的所有员工
```
SELECT empid, lastname
FROM HR.Employees
WHERE lastname LIKE N'[ABC]%';
```
④  [<字符-字符>]通配符：必须匹配指定范围内中的一个字符

　　下面请求返回lastname以字符A到E开头的所有员工：
```
SELECT empid, lastname
FROM HR.Employees
WHERE lastname LIKE N'[A-E]%';
```
⑤ [^<字符-字符>]通配符：不属于特定字符序列或范围内的任意单个字符

　　下面请求返回lastname不以A到E开头的所有员工：
```
SELECT empid, lastname
FROM HR.Employees
WHERE lastname LIKE N'[^A-E]%';
```
⑥ ESCAPE转义字符

　　如果搜索包含特殊通配符的字符串（例如'%','_','['、']'等），则必须使用转移字符。下面检查lastname列是否包含下划线：

```
SELECT empid, lastname
FROM HR.Employees
WHERE lastname LIKE N'%!_%' ESCAPE '!';
```

（10）两种转换值的函数：CAST和CONVERT

　　CAST和CONVERT都用于转换值的数据类型。
```
SELECT CAST(SYSDATETIME() AS DATE);
SELECT CONVERT(CHAR(8),CURRENT_TIMESTAMP,112);
```

##### 连接查询
数据库数据：
book表              
![book表](http://oy40mv0rj.bkt.clouddn.com/sql-join-01.png)
stu表
 ![book表](http://oy40mv0rj.bkt.clouddn.com/sql-join-01.png)

###### 内连接

1.1.等值连接：在连接条件中使用等于号(=)运算符比较被连接列的列值，其查询结果中列出被连接表中的所有列，包括其中的重复列。

1.2.不等值连接：在连接条件使用除等于运算符以外的其它比较运算符比较被连接的列的列值。这些运算符包括>、>=、<=、<、!>、!<和<>。

1.3.自然连接：在连接条件中使用等于(=)运算符比较被连接列的列值，但它使用选择列表指出查询结果集合中所包括的列，并删除连接表中的重复列。

内连接：内连接查询操作列出与连接条件匹配的数据行，它使用比较运算符比较被连接列的列值。
```
select * from book as a,stu as b where a.sutid = b.stuid

select * from book as a inner join stu as b on a.sutid = b.stuid
```

内连接可以使用上面两种方式，其中第二种方式的inner可以省略。

![](http://oy40mv0rj.bkt.clouddn.com/sql-join03.png)
其连接结果如上图，是按照a.stuid = b.stuid进行连接。

 

###### 外连接

2.1.左联接：是以左表为基准，将a.stuid = b.stuid的数据进行连接，然后将左表没有的对应项显示，右表的列为NULL
```
select * from book as a left join stu as b on a.sutid = b.stuid
```
![](http://oy40mv0rj.bkt.clouddn.com/sal-join04.png)
2.2.右连接：是以右表为基准，将a.stuid = b.stuid的数据进行连接，然以将右表没有的对应项显示，左表的列为NULL
```
select * from book as a right join stu as b on a.sutid = b.stuid
```
![](http://oy40mv0rj.bkt.clouddn.com/sql-join05.png)

2.3.全连接：完整外部联接返回左表和右表中的所有行。当某行在另一个表中没有匹配行时，则另一个表的选择列表列包含空值。如果表之间有匹配行，则整个结果集行包含基表的数据值。
```
select * from book as a full outer join stu as b on a.sutid = b.stuid
```
 ![](http://oy40mv0rj.bkt.clouddn.com/sql-join06.png)

###### 交叉连接

交叉连接：交叉联接返回左表中的所有行，左表中的每一行与右表中的所有行组合。交叉联接也称作笛卡尔积。
```
select * from book as a cross join stu as b order by a.id
```
![](http://oy40mv0rj.bkt.clouddn.com/sql-join07.png)
##### 子查询

#### 表达式
##### 派生表
##### 公用表表达式
##### 视图
##### 函数

####  集合运算
##### UNION 并集运算
##### INTERSECT 交集运算
##### EXCEPT 差集运算
##### 集合运算优先级
##### 使用表表达式避开不支持的逻辑查询处理

#### 透视、逆透视及分组
#### 数据修改
#### 事务和并发
#### 可编程对象
