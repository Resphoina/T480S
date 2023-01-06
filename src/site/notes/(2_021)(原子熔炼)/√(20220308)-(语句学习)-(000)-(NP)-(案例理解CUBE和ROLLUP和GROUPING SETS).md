---
{"dg-publish":true,"permalink":"/(2_021)(原子熔炼)/√(20220308)-(语句学习)-(000)-(NP)-(案例理解CUBE和ROLLUP和GROUPING SETS)/"}
---


# <font color=#DC143C>(20220308)-(语句学习)-(000)-(NP)-(案例理解CUBE和ROLLUP和GROUPING SETS)</font>
URL :: [Group By in SQL Server with CUBE, ROLLUP and GROUPING SETS Examples](https://www.mssqltips.com/sqlservertip/6315/group-by-in-sql-server-with-cube-rollup-and-grouping-sets-examples/)

| 入榜亮点 | 入榜输出 |
| ---- | ---- |
| \-   | \-   |


```
dataview
table without id 萃取函数, 萃取解法
where contains(TITLES, "")
```

```ad-note
title: 问题背景
collapse: open
The **GROUP BY** clause in SQL Server allows grouping of rows of a query.Generally, GROUP BY is used with an aggregate SQL Server function, such as <u>SUM</u>, <u>AVG</u>, etc. In addition,The GROUP BY can also be used with optional components such as Cube, Rollup and
Grouping Sets.<br/>
In this tip, I will demonstrate various ways of building a GROUP
BY along with output explained.<br/>
SQL Server中的GROUP BY子句允许对查询的行进行分组。一般来说，GROUP BY是和SQL Server的聚合函数一起使用的比如SUM，AVG等。此外GROUP BY也可以与可选的组件一起使用，如Cube、Rollup和Grouping Sets。<br/>
在这个提示中，我将演示建立GROUP BY的各种方法同时解释输出。
```

```ad-note
title: 解决思路
collapse: open
When building queries for reports, we often use the GROUP BY clause. There are also cases where having subtotals and totals as part of the output is helpful and this is where we will use the optional operators like: **CUBE, ROLLUP and GROUPING SETS**. These options are similar, but produce different results.<br/>
在为报告建立查询时，我们经常使用GROUP BY子句。在有些情况下，将小计和合计作为输出的一部分也是很有帮助的，这时我们会使用可选的运算符，如CUBE, ROLLUP 和GROUPING SETS。这些选项是相似的，但产生的结果不同。
```

## 01.Create Sample SQL Server Database and Data
First, we will create a sample database, table and insert some data for our examples.
```SQL
USE MASTER;
GO
CREATE DATABASE EMPTEST;
GO
USE EMPTEST;
GO
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CREATE TABLE]
CREATE TABLE EmpSalary(ID INT PRIMARY KEY IDENTITY(1, 1),
                       EmpName VARCHAR(200),
                       Department VARCHAR(100),
                       Category CHAR(1),
                       Salary MONEY);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[模拟数据]
INSERT EmpSalary
SELECT 'Bhavesh Patel', 'IT', 'A', $8000
UNION ALL
SELECT 'Alpesh Patel', 'Sales', 'A', $7000
UNION ALL
SELECT 'Kalpesh Thakor', 'IT', 'B', $5000
UNION ALL
SELECT 'Jay Shah', 'Sales', 'B', $4000
UNION ALL
SELECT 'Ram Nayak', 'IT', 'C', $3000
UNION ALL
SELECT 'Jay Shaw', 'Sales', 'C', $2000;
```

Here is the data we just created.
![|L](https://www.mssqltips.com/tipimages2/6315_group-by-cube-rollup-grouping-sets-sql-server.001-1.png)

## 02.SQL Server GROUP BY Example
Below is a simple Group By query we SUM the salary data. In the first query,
we group by Department and the second query we group by Department and Category.
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[部门汇总]
SELECT Department, SUM(Salary) AS Salary
FROM EmpSalary WITH(NOLOCK)
GROUP BY Department;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[部门类别汇总]
SELECT Department, Category, SUM(Salary) AS Salary
FROM EmpSalary
GROUP BY Department, Category;
```

Below are the results. The first query returns 2 rows by Department with
Salary totals, since there are 2 departments. The second query returns 6 rows by Department and Category with Salary totals, since there are 2 departments with 3 categories in each department.
![|L](https://www.mssqltips.com/tipimages2/6315_group-by-cube-rollup-grouping-sets-sql-server.002-1.png)

## 03.SQL Server GROUP BY with HAVING Example
In the next example, we use the same group by, but we limit the data using HAVING which filters the data. In the examples below, for the first query we only want to see Departments where the total equals 16000 and for the second where the Department and Category total equals 8000.

```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[部门汇总|过滤收入]
SELECT Department, SUM(Salary) AS Salary
FROM EmpSalary WITH(NOLOCK)
GROUP BY Department
HAVING SUM(Salary) = 16000;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[部门类别汇总|过滤收入]
SELECT Department, Category, SUM(Salary) AS Salary
FROM EmpSalary
GROUP BY Department, Category
HAVING SUM(Salary) = 8000;
```
![|L](https://www.mssqltips.com/tipimages2/6315_group-by-cube-rollup-grouping-sets-sql-server.003-1.png)

Below are the only 2 rows that meet the criteria. We can double check this
by looking at the query results from the first set of queries above.

## 04.SQL Server GROUP BY CUBE Example
This example allows us to show all combinations of the data. This includes
totals for the group combinations.
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[部门汇总]
SELECT Department, SUM(Salary) AS Salary
FROM EmpSalary
GROUP BY CUBE(Department);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[部门类别汇总]
SELECT Department, Category, SUM(Salary) AS Salary
FROM EmpSalary
GROUP BY CUBE(Department, Category);
```
![|L](https://www.mssqltips.com/tipimages2/6315_group-by-cube-rollup-grouping-sets-sql-server.004-1.png)
+ The first query results show the 2 Departments and the total, but also the grand total for these 2 Departments.
+ The second query results show us all of the combinations of Department and Category.
    + For example, we see IT (department) and A (category) and 16000 (total), then Sales (department) and A (category) and 7000 (total) and then NULL (both departments) and A (category) and 15000 (total).
    + In the chart, I break down the different groupings that are part of this second query.

## 05.SQL Server GROUP BY ROLLUP Example
This is similar to the Group By Cube, but you will see the output is slightly
different where we don't get as many rows returned for the second query.
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[部门汇总]
SELECT Department, SUM(Salary) AS Salary
FROM EmpSalary WITH(NOLOCK)
GROUP BY ROLLUP(Department);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[部门类别汇总]
SELECT Department, Category, SUM(Salary) AS Salary
FROM EmpSalary WITH(NOLOCK)
GROUP BY ROLLUP(Department, Category);
```
![|L](https://www.mssqltips.com/tipimages2/6315_group-by-cube-rollup-grouping-sets-sql-server.005-1.png)
+ We can see the first query results are the same as Group By Rollup example, but the second query only returns 9 rows instead of 12 that we got in the Group By Rollup query.
+ The second query does the rollup first by Department and then by Category, which is different from the Group By Cube which did the rollup in both directions. This allows us to get subtotals for each Department and an overall total for all Departments.

We could change the second query, as shown below, to first rollup by Category and then Department.We can see the results for the second query now do the grouping based on Category and then Department. And we still get the subtotals and totals.
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[部门汇总]
SELECT Department, SUM(Salary) AS salary
FROM EmpSalary
GROUP BY ROLLUP(Department);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[对调部门类别顺序]
SELECT Department, Category, SUM(Salary) AS salary
FROM EmpSalary
GROUP BY ROLLUP(Category, Department);
```
![|L](https://www.mssqltips.com/tipimages2/6315_group-by-cube-rollup-grouping-sets-sql-server.006-2.png)

## 06.SQL Server GROUP BY ROLLUP with GROUPING\_ID Example
Another option is to use[GROUPING\_ID](https://docs.microsoft.com/en-us/sql/t-sql/functions/grouping-id-transact-sql?view=sql-server-ver15) as part of the result set to show each group.

```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[GROUPING_ID]
SELECT Department,
       Category,
       SUM(Salary) AS Salary,
       GROUPING_ID(Category, Department) AS GroupingID
FROM EmpSalary
GROUP BY ROLLUP(Category, Department);
```

We can see we have the same results as above, but now we have a grouping value for each of these groups.
![|L](https://www.mssqltips.com/tipimages2/6315_group-by-cube-rollup-grouping-sets-sql-server.007-1.png)

## 07.SQL Server GROUP BY GROUPING SETS Example
With grouping sets, we can determine how we want the data to be put together.
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[GROUPING SETS]
SELECT Department,
       Category,
       SUM(Salary) AS Salary
FROM EmpSalary WITH(NOLOCK)
GROUP BY GROUPING SETS(Category, Department, (Category, Department), ());
```

Below we can see we did a group for Category, another group for Department, another group for Category and Department and the last group for NULL.

![|L](https://www.mssqltips.com/tipimages2/6315_group-by-cube-rollup-grouping-sets-sql-server.008-1.png)

Here is another example by Department and Category and an overall group for NULL.
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[GROUPING SETS]
SELECT Department,
       Category,
       SUM(Salary) AS Salary
FROM EmpSalary WITH(NOLOCK)
GROUP BY GROUPING SETS((Department, Category), ());
```

![|L](https://www.mssqltips.com/tipimages2/6315_group-by-cube-rollup-grouping-sets-sql-server.008-2.png)

We could also take this a step further and use CUBE and ROLLUP for the different grouping sets.
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[GROUPING SETS & CUBE & ROLLUP]
SELECT Department,
       Category,
       SUM(Salary) AS Salary
FROM EmpSalary WITH(NOLOCK)
GROUP BY GROUPING SETS(CUBE(Department, Category),
                       ROLLUP(Department, Category));
```

Here is the output.
![|L](https://www.mssqltips.com/tipimages2/6315_group-by-cube-rollup-grouping-sets-sql-server.009-1.png)

## 08.Summary
As per the reporting purpose for preparing a summarized output, we can use optional operators such as CUBE, ROLLUP, and GROUPING SETS in the query. GROUPING SETS is a controllable and scalable option, so I prefer to using it in lieu of ROLLUP and CUBE.(根据报告的目的，我们可以在查询中使用可选的运算符，如CUBE, ROLLUP, GROUPING SETS。萃取输出:: **_GROUPING SETS是一个可控的、可扩展的选项，所以我更愿意用它来代替ROLLUP和CUBE_**。)