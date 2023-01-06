---
{"dg-publish":true,"permalink":"/6-000/6-002/20220301-602-sql-prompt-snippets/"}
---


# <font color=#DC143C>(20220301)-(清单宝典)-(602)-(手册)-(SQL PROMPT SNIPPETS)</font>
相关内链:: 
相关人员:: 
相关过程:: 
问题分析:: 
我的处理:: 
操作简述:: 

```
dataview
table without id 入榜亮点, 入榜输出
where contains(TITLES, "")
```

| 问题分析 | 我的处理 | 操作简述 |
| ---- | ---- | ---- |
| \-   | \-   | \-   |


```toc
```

# SQL PROMPT SNIPPETS

<hr style="border:3px solid ForestGreen"> </hr>

## MSYS

<hr style="border:2px solid DeepSkyBlue"> </hr>

### 表上下游——MSYS_shangxiayou
#### 逻辑压缩
基于过程脚本的写法规律提取`tbTableIn`、`tbTableOut`、`tbTableTemp`三个表，依照以下的查询语句实现：
+ 从过程找表
+ 从表找过程
+ 从表找过程和作业信息
+ 从表找多级调用或者多级源头的信息

#### 注意事项
+ tbTableIn、tbTableOut、tbTableTemp当天修改的部分无法查询
+ 动态拼接无法查询

#### 新版本——20220705
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[案例思想]
SET NOEXEC ON;
SELECT CHARINDEX('12345', '123456789', 1),/*前缀相同-顺序*/
       CHARINDEX(REVERSE('12345'), REVERSE('123456789')),/*前缀相同-倒序*/
       CHARINDEX(REVERSE('12345'), REVERSE('123A456789')),/*前缀不同-顺序*/
       CHARINDEX('12345', '12345', 1),/*完全相同-顺序*/
       CHARINDEX(REVERSE('12345'), REVERSE('12345'), 1),/*完全相同-倒序*/
       CHARINDEX('12345', 'AANNCC123456789', 1),/*满足出现-顺序*/
       CHARINDEX(REVERSE('12345'), REVERSE('AANNCC123456789'), 1)/*满足出现-倒序*/
SET NOEXEC OFF;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[变量赋值]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @OBJECT NVARCHAR(1000);
DECLARE @CHOICE INT;
DECLARE @FULLMATCH INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @OBJECT = N'shx_qh_history_proc';
--SET @OBJECT = N'basic_proc';
SET @OBJECT = N'odsdbfq.dbo.moutdrpt';
SET @CHOICE = 2;
SET @FULLMATCH = 0;/*|1精确|0模糊|*/
SET @FULLMATCH = 1;/*|1精确|0模糊|*/
--1:过程——用什么表生成
--2:过程——生成了什么表
--3:过程——生成的临时表
--4:表——用什么表生成
--5:表——什么过程用到了表
--6:表——用什么表生成进而找到过程
--7:表——什么过程用到了表进而找到过程
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从过程找表]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询过程的源头表——过程——用什么表生成]
IF @CHOICE = 1
BEGIN
     --案例:EveryDayRenew_stsale_Proc_new
     SELECT 数据库, 对象类型, 对象名, 表名
     FROM odsdbbi.dbo.tbTableIn WITH(NOLOCK)
     WHERE(CHARINDEX(@OBJECT, 对象名, 1) > 0)
     AND  (@FULLMATCH = 0 OR (@FULLMATCH = 1 AND (CHARINDEX(REVERSE(@OBJECT), REVERSE(对象名), 1) = CHARINDEX(@OBJECT, 对象名, 1))))
     ORDER BY 数据库, 对象类型;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询过程的生成表——过程——生成了什么表]
IF @CHOICE = 2
BEGIN
     SELECT 数据库, 对象类型, 对象名, 表名
     FROM odsdbbi.dbo.tbTableOut WITH(NOLOCK)
     WHERE (CHARINDEX(@OBJECT, 对象名, 1) > 0)
     AND   (@FULLMATCH = 0 OR (@FULLMATCH = 1 AND (CHARINDEX(REVERSE(@OBJECT), REVERSE(对象名), 1) = CHARINDEX(@OBJECT, 对象名, 1))))
     ORDER BY 数据库, 对象类型;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询过程的临时表——过程——生成的临时表]
IF @CHOICE = 3
BEGIN
     SELECT 数据库, 对象类型, 对象名, 临时表
     FROM odsdbbi.dbo.tbTableTemp WITH(NOLOCK)
     WHERE (CHARINDEX(@OBJECT, 对象名, 1) > 0)
     AND   (@FULLMATCH = 0 OR (@FULLMATCH = 1 AND (CHARINDEX(REVERSE(@OBJECT), REVERSE(对象名), 1) = CHARINDEX(@OBJECT, 对象名, 1))))
     ORDER BY 数据库, 对象类型;
END
--CREATE CLUSTERED INDEX IDX_OBJECT ON tbTableTemp(对象名);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从表找过程]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的源头过程——表——用什么表生成]
IF @CHOICE = 4
BEGIN
     SELECT 表名, 数据库, 对象类型, 对象名
     FROM odsdbbi.dbo.tbTableOut WITH(NOLOCK)
     WHERE (CHARINDEX(@OBJECT, 表名, 1) > 0)
     AND   (@FULLMATCH = 0 OR (@FULLMATCH = 1 AND (CHARINDEX(REVERSE(@OBJECT), REVERSE(表名), 1) = CHARINDEX(@OBJECT, 表名, 1))))
     ORDER BY 对象类型 DESC, 数据库;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的调用过程——表——什么过程用到了表]
IF @CHOICE = 5
BEGIN
     SELECT 表名, 数据库, 对象类型, 对象名
     FROM odsdbbi.dbo.tbTableIn WITH(NOLOCK)
     WHERE (CHARINDEX(@OBJECT, 表名, 1) > 0)
     AND   (@FULLMATCH = 0 OR (@FULLMATCH = 1 AND (CHARINDEX(REVERSE(@OBJECT), REVERSE(表名), 1) = CHARINDEX(@OBJECT, 表名, 1))))
     ORDER BY 对象类型 DESC, 数据库;
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从表找过程和作业信息]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的源头过程和作业信息——表——用什么表生成进而找到过程]
IF @CHOICE = 6
BEGIN
     SELECT 表名, m1.数据库, 对象类型, 对象名, 作业名, 步骤号, 步骤名
     FROM odsdbbi.dbo.tbTableOut AS m1 WITH(NOLOCK)
     LEFT OUTER JOIN odsdbbi.dbo.tbJob AS m2 WITH(NOLOCK)
     ON  m1.数据库 = m2.数据库
     AND m1.对象名 = m2.过程名
     WHERE (CHARINDEX(@OBJECT, 表名, 1) > 0)
     AND   (@FULLMATCH = 0 OR (@FULLMATCH = 1 AND (CHARINDEX(REVERSE(@OBJECT), REVERSE(表名), 1) = CHARINDEX(@OBJECT, 表名, 1))))
     ORDER BY 对象类型 DESC, 数据库, 作业名, 步骤号;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的调用过程和作业信息——表——什么过程用到了表进而找到过程]
IF @CHOICE = 7
BEGIN
     SELECT 表名, m1.数据库, 对象类型, 对象名, 作业名, 步骤号, 步骤名
     FROM odsdbbi.dbo.tbTableIn AS m1 WITH(NOLOCK)
     LEFT OUTER JOIN odsdbbi.dbo.tbJob AS m2 WITH(NOLOCK)
     ON  m1.数据库 = m2.数据库
     AND m1.对象名 = m2.过程名
     WHERE (CHARINDEX(@OBJECT, 表名, 1) > 0)
     AND   (@FULLMATCH = 0 OR (@FULLMATCH = 1 AND (CHARINDEX(REVERSE(@OBJECT), REVERSE(表名), 1) = CHARINDEX(@OBJECT, 表名, 1))))
     ORDER BY 对象类型 DESC, 数据库, 作业名, 步骤号;
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从表找多级调用或者多级源头的信息]
SET NOEXEC ON
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[表调用多级查询]
--∷∷∷∷∷∷[完整显示格式|一级]
EXEC odsdbbi.dbo.tbApply @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'full';
--∷∷∷∷∷∷[简约显示格式|一级]
EXEC odsdbbi.dbo.tbApply @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'simple';
--∷∷∷∷∷∷[简约显示格式|二级]
EXEC odsdbbi.dbo.tbApply @table_name = 'odsdb.dbo.m_stsale', @level = '2', @format = 'simple';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[表源头多级查询]
--∷∷∷∷∷∷[完整显示格式|一级]
EXEC odsdbbi.dbo.tbSource @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'full';
--∷∷∷∷∷∷[简约显示格式|一级]
EXEC odsdbbi.dbo.tbSource @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'simple';
--∷∷∷∷∷∷[简约显示格式|二级]
EXEC odsdbbi.dbo.tbSource @table_name = 'odsdb.dbo.m_stsale', @level = '2', @format = 'simple';
SET NOEXEC OFF
```

#### 老版本——20220705
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @OBJECT NVARCHAR(1000);
DECLARE @CHOICE INT;
DECLARE @FULLMATCH INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @OBJECT = N'EveryDayRenew_stsale_Proc';
SET @OBJECT = N'odsdbfq.dbo.goods';
SET @CHOICE = 1;
SET @FULLMATCH = 0;/*|1精确|0模糊|*/
SET @FULLMATCH = 1;/*|1精确|0模糊|*/
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从过程找表]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询过程的源头表——过程——用什么表生成]
IF @CHOICE = 1
BEGIN
     --案例:EveryDayRenew_stsale_Proc_new
     SELECT 数据库, 对象类型, 对象名, 表名
     FROM odsdbbi.dbo.tbTableIn WITH(NOLOCK)
     WHERE (CASE WHEN @FULLMATCH = 0
                 THEN CHARINDEX(@OBJECT, 对象名, 1)
                 ELSE CHARINDEX(REVERSE(@OBJECT), REVERSE(对象名), 1)
                 END) = 1
     ORDER BY 数据库, 对象类型;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询过程的生成表——过程——生成了什么表]
IF @CHOICE = 2
BEGIN
     SELECT 数据库, 对象类型, 对象名, 表名
     FROM odsdbbi.dbo.tbTableOut WITH(NOLOCK)
     WHERE (CASE WHEN @FULLMATCH = 0
                 THEN CHARINDEX(@OBJECT, 对象名, 1)
                 ELSE CHARINDEX(REVERSE(@OBJECT), REVERSE(对象名), 1)
                 END) = 1
     ORDER BY 数据库, 对象类型;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询过程的临时表——过程——生成的临时表]
IF @CHOICE = 3
BEGIN
     SELECT 数据库, 对象类型, 对象名, 临时表
     FROM odsdbbi.dbo.tbTableTemp WITH(NOLOCK)
     WHERE (CASE WHEN @FULLMATCH = 0
                 THEN CHARINDEX(@OBJECT, 对象名, 1)
                 ELSE CHARINDEX(REVERSE(@OBJECT), REVERSE(对象名), 1)
                 END) = 1
     ORDER BY 数据库, 对象类型;
END
--CREATE CLUSTERED INDEX IDX_OBJECT ON tbTableTemp(对象名);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从表找过程]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的源头过程——表——用什么表生成]
IF @CHOICE = 4
BEGIN
     SELECT 表名, 数据库, 对象类型, 对象名
     FROM odsdbbi.dbo.tbTableOut WITH(NOLOCK)
     WHERE (CASE WHEN @FULLMATCH = 0
                 THEN CHARINDEX(@OBJECT, 表名, 1)
                 ELSE CHARINDEX(REVERSE(@OBJECT), REVERSE(表名), 1)
                 END) = 1
     ORDER BY 对象类型 DESC, 数据库;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的调用过程——表——什么过程用到了表]
IF @CHOICE = 5
BEGIN
     SELECT 表名, 数据库, 对象类型, 对象名
     FROM odsdbbi.dbo.tbTableIn WITH(NOLOCK)
     WHERE (CASE WHEN @FULLMATCH = 0
                 THEN CHARINDEX(@OBJECT, 表名, 1)
                 ELSE CHARINDEX(REVERSE(@OBJECT), REVERSE(表名), 1)
                 END) = 1
     ORDER BY 对象类型 DESC, 数据库;
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从表找过程和作业信息]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的源头过程和作业信息——表——用什么表生成进而找到过程]
IF @CHOICE = 6
BEGIN
     SELECT 表名, m1.数据库, 对象类型, 对象名, 作业名, 步骤号, 步骤名
     FROM odsdbbi.dbo.tbTableOut AS m1 WITH(NOLOCK)
     LEFT OUTER JOIN odsdbbi.dbo.tbJob AS m2 WITH(NOLOCK)
     ON  m1.数据库 = m2.数据库
     AND m1.对象名 = m2.过程名
     WHERE (CASE WHEN @FULLMATCH = 0
                 THEN CHARINDEX(@OBJECT, 表名, 1)
                 ELSE CHARINDEX(REVERSE(@OBJECT), REVERSE(表名), 1)
                 END) = 1
     ORDER BY 对象类型 DESC, 数据库, 作业名, 步骤号;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的调用过程和作业信息——表——什么过程用到了表进而找到过程]
IF @CHOICE = 7
BEGIN
     SELECT 表名, m1.数据库, 对象类型, 对象名, 作业名, 步骤号, 步骤名
     FROM odsdbbi.dbo.tbTableIn AS m1 WITH(NOLOCK)
     LEFT OUTER JOIN odsdbbi.dbo.tbJob AS m2 WITH(NOLOCK)
     ON  m1.数据库 = m2.数据库
     AND m1.对象名 = m2.过程名
     WHERE (CASE WHEN @FULLMATCH = 0
                 THEN CHARINDEX(@OBJECT, 表名, 1)
                 ELSE CHARINDEX(REVERSE(@OBJECT), REVERSE(表名), 1)
                 END) = 1
     ORDER BY 对象类型 DESC, 数据库, 作业名, 步骤号;
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从表找多级调用或者多级源头的信息]
SET NOEXEC ON
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[表调用多级查询]
--∷∷∷∷∷∷[完整显示格式|一级]
EXEC odsdbbi.dbo.tbApply @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'full';
--∷∷∷∷∷∷[简约显示格式|一级]
EXEC odsdbbi.dbo.tbApply @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'simple';
--∷∷∷∷∷∷[简约显示格式|二级]
EXEC odsdbbi.dbo.tbApply @table_name = 'odsdb.dbo.m_stsale', @level = '2', @format = 'simple';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[表源头多级查询]
--∷∷∷∷∷∷[完整显示格式|一级]
EXEC odsdbbi.dbo.tbSource @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'full';
--∷∷∷∷∷∷[简约显示格式|一级]
EXEC odsdbbi.dbo.tbSource @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'simple';
--∷∷∷∷∷∷[简约显示格式|二级]
EXEC odsdbbi.dbo.tbSource @table_name = 'odsdb.dbo.m_stsale', @level = '2', @format = 'simple';
SET NOEXEC OFF
```

#### 初始版——20220705
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从过程找表]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询过程的源头表——用什么表生成]
--案例:EveryDayRenew_stsale_Proc_new
SELECT 数据库, 对象类型, 对象名, 表名
FROM odsdbbi.dbo.tbTableIn WITH(NOLOCK)
WHERE 对象名 = 'EveryDayRenew_stsale_Proc_new'--过程名
ORDER BY 数据库, 对象类型;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询过程的生成表——生成了什么表]
SELECT 数据库, 对象类型, 对象名, 表名
FROM odsdbbi.dbo.tbTableOut WITH(NOLOCK)
WHERE 对象名 = 'EveryDayRenew_stsale_Proc_new'
ORDER BY 数据库, 对象类型;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询过程的临时表——生成的临时表]
SELECT 数据库, 对象类型, 对象名, 临时表
FROM odsdbbi.dbo.tbTableTemp WITH(NOLOCK)
WHERE 对象名 = 'EveryDayRenew_stsale_Proc_new'
ORDER BY 数据库, 对象类型;
--CREATE CLUSTERED INDEX IDX_OBJECT ON tbTableTemp(对象名);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从表找过程]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的源头过程——用什么表生成]
SELECT 表名, 数据库, 对象类型, 对象名
FROM odsdbbi.dbo.tbTableOut WITH(NOLOCK)
WHERE 表名 = 'odsdb.dbo.m_stsale'
ORDER BY 对象类型 DESC, 数据库;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的调用过程——什么过程用到了表]
SELECT 表名, 数据库, 对象类型, 对象名
FROM odsdbbi.dbo.tbTableIn WITH(NOLOCK)
WHERE 表名 = 'odsdb.dbo.m_stsale'
ORDER BY 对象类型 DESC, 数据库;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从表找过程和作业信息]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的源头过程和作业信息——用什么表生成进而找到过程]
SELECT 表名, m1.数据库, 对象类型, 对象名, 作业名, 步骤号, 步骤名
FROM odsdbbi.dbo.tbTableOut AS m1 WITH(NOLOCK)
LEFT OUTER JOIN odsdbbi.dbo.tbJob AS m2 WITH(NOLOCK)
ON  m1.数据库 = m2.数据库
AND m1.对象名 = m2.过程名
WHERE 表名 = 'odsdb.dbo.m_stsale'
ORDER BY 对象类型 DESC, 数据库, 作业名, 步骤号;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查询表的调用过程和作业信息——什么过程用到了表进而找到过程]
SELECT 表名, m1.数据库, 对象类型, 对象名, 作业名, 步骤号, 步骤名
FROM odsdbbi.dbo.tbTableIn AS m1 WITH(NOLOCK)
LEFT OUTER JOIN odsdbbi.dbo.tbJob AS m2 WITH(NOLOCK)
ON  m1.数据库 = m2.数据库
AND m1.对象名 = m2.过程名
WHERE 表名 = 'odsdb.dbo.m_stsale'
ORDER BY 对象类型 DESC, 数据库, 作业名, 步骤号;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[从表找多级调用或者多级源头的信息]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[表调用多级查询]
--∷∷∷∷∷∷[完整显示格式|一级]
EXEC odsdbbi.dbo.tbApply @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'full';
--∷∷∷∷∷∷[简约显示格式|一级]
EXEC odsdbbi.dbo.tbApply @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'simple';
--∷∷∷∷∷∷[简约显示格式|二级]
EXEC odsdbbi.dbo.tbApply @table_name = 'odsdb.dbo.m_stsale', @level = '2', @format = 'simple';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[表源头多级查询]
--∷∷∷∷∷∷[完整显示格式|一级]
EXEC odsdbbi.dbo.tbSource @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'full';
--∷∷∷∷∷∷[简约显示格式|一级]
EXEC odsdbbi.dbo.tbSource @table_name = 'odsdb.dbo.m_stsale', @level = '1', @format = 'simple';
--∷∷∷∷∷∷[简约显示格式|二级]
EXEC odsdbbi.dbo.tbSource @table_name = 'odsdb.dbo.m_stsale', @level = '2', @format = 'simple';
```

### 查表作业——MSYS_biaozuoye
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[赋值变量]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @STRING NVARCHAR(1000);
DECLARE @CHOICE NVARCHAR(1000);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SELECT @STRING = N'XHQ';
SELECT @CHOICE = N'J';
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[存储过程查询]
SET NOEXEC ON;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[sys.procedures|明确过程]
SELECT [name] AS procedure_name,
       [object_id],
       [create_date],
       [modify_date]
FROM [sys].[procedures] WITH(NOLOCK)
WHERE [NAME] = @STRING;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[sys.sql_modules|过程定义]
SELECT TOP 3
       [object_id],
       [definition]
FROM [sys].[sql_modules] WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[sys.procedures]&[sys.sql_modules]
SELECT a.[name] AS procedure_name,
       a.[object_id],
       a.[create_date],
       a.[modify_date],
       b.[definition]
FROM [sys].[procedures]        AS a WITH(NOLOCK)
INNER JOIN [sys].[sql_modules] AS b WITH(NOLOCK)
ON a.[object_id] = b.[object_id]
WHERE a.[name] = @STRING;
SET NOEXEC OFF;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[表名查找过程|TABLENAME→PROCEDURE]
IF @CHOICE = 'P'
BEGIN
     SELECT DISTINCT
            a.[name] AS PROCNAME,
            a.XTYPE,
            a.CRDATE,
            b.[text] AS SQLCMD
     FROM sysobjects        AS a WITH(NOLOCK)
     INNER JOIN syscomments AS b WITH(NOLOCK)
     ON a.ID = b.ID
     WHERE xtype = 'p'
     AND   CHARINDEX(@STRING, b.[text], 1) > 0
     ORDER BY a.CRDATE DESC;
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[过程查找作业|PROCEDURE→SYSJOBS]
IF @CHOICE = 'J'
BEGIN
     SELECT a.name AS SYSJOBS_NAME,
            b.step_name AS SYSJOBSTEPS_NAME,
            b.command AS SQLTEXT
     FROM msdb.dbo.sysjobs AS a WITH(NOLOCK)
     INNER JOIN msdb.dbo.sysjobsteps AS b WITH(NOLOCK)
     ON a.job_id = b.job_id
     WHERE CHARINDEX(@STRING, b.command, 1) > 0
     ORDER BY a.name ASC, b.step_id ASC;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[YHM_JOBDETAIL]
SET NOEXEC ON;
SELECT *
FROM odsdb.dbo.YHM_JOBDETAIL WITH(NOLOCK)
WHERE CHARINDEX(@STRING, COMMAND, 1) > 0;
SET NOEXEC OFF;
```

### 报错作业——MSYS_baocuozuoye
#### 新版本—20220302
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[执行过程]
EXEC odsdb.dbo.P_SYSJOB_FAILED;
--EXEC pp_sysjob_failed '61|5_161|5_162|28_HN';
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[变量设置]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @BelongTo_SINGLE NVARCHAR(MAX);
DECLARE @TURN_ON INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @BelongTo_SINGLE = N'徐海权';
SET @TURN_ON = 1;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[呈现方式]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[特定对象]
IF @TURN_ON = 1
BEGIN
     WITH STRING_SPLIT_VALUE
     AS (SELECT VALUE
         FROM STRING_SPLIT(@BelongTo_SINGLE, '|'))
     SELECT *
     FROM odsdb.dbo.tb_sysjob_fail WITH(NOLOCK)
     WHERE BelongTo IN(SELECT * FROM STRING_SPLIT_VALUE WITH(NOLOCK))
     ORDER BY ServerIP, BelongTo;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[特定部门]
IF @TURN_ON = 2
BEGIN
     WITH STRING_SPLIT_VALUE
     AS (SELECT USERNAME
         FROM XTB_BI_TEAMMATE_ABBR WITH(NOLOCK)
         WHERE BOSS = @BelongTo_SINGLE)
     SELECT *
     FROM odsdb.dbo.tb_sysjob_fail WITH(NOLOCK)
     WHERE BelongTo IN(SELECT * FROM STRING_SPLIT_VALUE WITH(NOLOCK))
     ORDER BY ServerIP, BelongTo;
END
```

#### 老版本—20220302
```SQL
--======================[61]将报错数据取出来
SELECT '61' AS ip, a.name, SUBSTRING(a.name, 6, CHARINDEX('】', a.name, 1) - 5) xitong, LEFT(a.name, 5) AS fuzeren, [message], 
CONVERT(DATETIME, CONVERT(VARCHAR(12), run_date)) run_date, GETDATE() gengxin_time, 
CASE WHEN LEN(description) <= 13 THEN '' ELSE RIGHT(a.description, LEN(RTRIM(description)) - 13)END MARK, description --,step_name
FROM msdb.dbo.sysjobs a WITH(NOLOCK)--61
INNER JOIN msdb.dbo.sysjobhistory b WITH(NOLOCK)--61
ON a.job_id = b.job_id
WHERE run_date >= CONVERT(CHAR(8), GETDATE()-1, 112)
--run_date >= CONVERT(CHAR(8), GETDATE() - 1 + 0.5, 112)
AND message LIKE '%失败%'
AND step_name NOT LIKE '%作业结果%'
AND [enabled] = 1--取值正在运行的作业,0是禁用的作业
AND CHARINDEX('】', a.name, 1) >= 5
AND CASE WHEN run_date = CONVERT(CHAR(8), CURRENT_TIMESTAMP - 1, 112) 
                    THEN CASE WHEN LEFT(run_time, 1) IN('1', '2') AND LEFT(run_time, 2) > 14 THEN 1 ELSE 0 END
                    ELSE 1 END = 1
UNION ALL
--======================[161]将报错数据取出来
SELECT '161' AS ip, a.name, SUBSTRING(a.name, 6, CHARINDEX('】', a.name, 1) - 5) xitong, LEFT(a.name, 5) AS fuzeren, [message], 
CONVERT(DATETIME, CONVERT(VARCHAR(12), run_date)) run_date, GETDATE() gengxin_time, 
CASE WHEN LEN(description) <= 13 THEN '' ELSE RIGHT(a.description, LEN(RTRIM(description)) - 13)END MARK, description --,step_name
FROM DATABASE_5_161.msdb.dbo.sysjobs a WITH(NOLOCK)--161
INNER JOIN DATABASE_5_161.msdb.dbo.sysjobhistory b WITH(NOLOCK)--161
ON a.job_id = b.job_id
WHERE run_date >= CONVERT(CHAR(8), GETDATE() - 1, 112)
--run_date >= CONVERT(CHAR(8), GETDATE() - 1 + 0.5, 112)
AND message LIKE '%失败%'
AND step_name NOT LIKE '%作业结果%'
AND [enabled] = 1--取值正在运行的作业,0是禁用的作业
AND CHARINDEX('】', a.name, 1) >= 5
AND CASE WHEN run_date = CONVERT(CHAR(8), CURRENT_TIMESTAMP - 1, 112) 
                    THEN CASE WHEN LEFT(run_time, 1) IN('1', '2') AND LEFT(run_time, 2) > 14 THEN 1 ELSE 0 END
                    ELSE 1 END = 1
UNION ALL
--======================[162]将报错数据取出来
SELECT '162' AS ip, a.name, SUBSTRING(a.name, 6, CHARINDEX('】', a.name, 1) - 5) xitong, LEFT(a.name, 5) AS fuzeren, [message], 
CONVERT(DATETIME, CONVERT(VARCHAR(12), run_date)) run_date, GETDATE() gengxin_time, 
CASE WHEN LEN(description) <= 13 THEN '' ELSE RIGHT(a.description, LEN(RTRIM(description)) - 13)END MARK, description --,step_name
FROM DATABASE_5_162.msdb.dbo.sysjobs a WITH(NOLOCK)--162
INNER JOIN DATABASE_5_162.msdb.dbo.sysjobhistory b WITH(NOLOCK)--162
ON a.job_id = b.job_id
WHERE run_date >= CONVERT(CHAR(8), GETDATE() - 1, 112)
--run_date >= CONVERT(CHAR(8), GETDATE() - 1 + 0.5, 112)
AND message LIKE '%失败%'
AND step_name NOT LIKE '%作业结果%'
AND [enabled] = 1--取值正在运行的作业,0是禁用的作业
AND CHARINDEX('】', a.name, 1) >= 5
AND CASE WHEN run_date = CONVERT(CHAR(8), CURRENT_TIMESTAMP - 1, 112) 
                    THEN CASE WHEN LEFT(run_time, 1) IN('1', '2') AND LEFT(run_time, 2) > 14 THEN 1 ELSE 0 END
                    ELSE 1 END = 1
```

<hr style="border:2px solid DeepSkyBlue"> </hr>

### 成功作业——MSYS_chenggongzuoye
```SQL
USE msdb;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @JOBNAME NVARCHAR(MAX);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SEARCH]
SET @JOBNAME = N'流水';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[RESULT]
SELECT b.name AS JobName,
       CONVERT(VARCHAR, DATEADD(S, (run_time / 10000) * 60 * 60/*hours*/
                                + ((run_time - (run_time / 10000) * 10000) / 100) * 60/*mins*/
                                + (run_time - (run_time / 100) * 100),/*secs*/
                                CONVERT(DATETIME, RTRIM(run_date), 113)),
               100) AS TimeRun,
       CASE WHEN ENABLED = 1 THEN 'Enabled' ELSE 'Disabled' END AS JobStatus,
       CASE WHEN a.run_status = 0 THEN 'FAILED'
            WHEN a.run_status = 1 THEN 'SUCCEEDED'
            WHEN a.run_status = 2 THEN 'RETRY'
            WHEN a.run_status = 3 THEN 'CANCELLED'
       ELSE 'UNKNOWN' END JobOutcome,
       b.job_id
FROM msdb.dbo.sysjobhistory AS a WITH(NOLOCK)
INNER JOIN msdb.dbo.sysjobs AS b WITH(NOLOCK)
ON a.job_id = b.job_id
WHERE a.step_id = 0
AND DATEADD(S, (run_time / 10000) * 60 * 60/*hours*/
            + ((run_time - (run_time / 10000) * 10000) / 100) * 60/*mins*/
            + (run_time - (run_time / 100) * 100),/*secs*/
            CONVERT(DATETIME, RTRIM(run_date), 113)) >= DATEADD(DAY, -1, GETDATE())
AND CASE WHEN LEN(@JOBNAME) > 0 THEN CHARINDEX(@JOBNAME, b.name, 1) ELSE 0 END > 1
ORDER BY run_date DESC, run_time DESC;
```

<hr style="border:2px solid DeepSkyBlue"> </hr>

### 在跑脚本——MSYS_RUNNING
#### 新版本——20220720
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[在跑脚本]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @LOGIN_NAME NVARCHAR(MAX);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @LOGIN_NAME = N'';
SELECT u.USERNAME,
       p.SESSION_ID, /*p.[HOST_NAME],*/ p.[PROGRAM_NAME], /*p.CLIENT_INTERFACE_NAME,*/ p.LOGIN_NAME, p.LOGIN_TIME, /*p.NT_DOMAIN,*/ /*p.NT_USER_NAME,*/
       c.DATABASENAME, /*c.OBJNAME,*/ c.QUERY,
       b.CLIENT_NET_ADDRESS,
       a.START_TIME AS '开始时间', a.SESSION_ID, /*a.REQUEST_ID,*/ a.BLOCKING_SESSION_ID AS '阻塞ID', a.[STATUS] AS '状态', a.COMMAND AS '命令类型',
       DB_NAME(a.database_id) AS '数据库名', a.READS AS '物理读次数', a.WRITES AS '写次数', a.LOGICAL_READS AS '逻辑读次数',
       a.ROW_COUNT AS '返回结果行数', a.WAIT_TYPE AS '等待资源类型', a.WAIT_TIME AS '等待时间', a.WAIT_RESOURCE AS '等待的资源',
       b.CLIENT_NET_ADDRESS, b.LOCAL_NET_ADDRESS
FROM SYS.DM_EXEC_REQUESTS AS a WITH(NOLOCK)
INNER JOIN SYS.DM_EXEC_CONNECTIONS AS b WITH(NOLOCK)
ON a.session_id = b.session_id
INNER JOIN SYS.DM_EXEC_SESSIONS AS p WITH(NOLOCK)
ON b.session_id = p.session_id
LEFT OUTER JOIN .dbo.user_dept_detail AS u WITH(NOLOCK)
ON (CASE WHEN CHARINDEX('_', p.LOGIN_NAME, 1) > 0 THEN LEFT(p.LOGIN_NAME, LEN(p.LOGIN_NAME) - 3) ELSE p.LOGIN_NAME END) = u.userid
CROSS APPLY(SELECT DB_NAME(DBID) AS DATABASENAME,
                   OBJECT_ID(OBJECTID) AS OBJNAME,
                   ISNULL((SELECT TEXT AS [processing-instruction(definition)]
                           FROM SYS.DM_EXEC_SQL_TEXT(b.MOST_RECENT_SQL_HANDLE)
                           FOR XML PATH(''), TYPE), '') AS QUERY
            FROM SYS.DM_EXEC_SQL_TEXT(b.MOST_RECENT_SQL_HANDLE)) c
WHERE a.SESSION_ID <> @@SPID
AND   a.SESSION_ID > 50
AND   (CASE WHEN LEN(@LOGIN_NAME) > 0 THEN @LOGIN_NAME ELSE p.LOGIN_NAME END) = p.LOGIN_NAME
SET NOEXEC ON
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[24小时内执行过的脚本]
GOTO MRX_SKIP;
SELECT a.last_execution_time AS TimeExecute, b.[text] AS Script
FROM SYS.DM_EXEC_QUERY_STATS AS a WITH(NOLOCK)
CROSS APPLY SYS.DM_EXEC_SQL_TEXT(a.sql_handle) AS b
ORDER BY a.last_execution_time DESC;
MRX_SKIP:
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[详细查询]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[当前使用库名]
/*EXEC zhh_useractive--查询运行中的脚本
EXEC zhh_system_sql--查询运行中的脚本*/
DECLARE @dbname VARCHAR(200);
SELECT @dbname = [name]
FROM master.dbo.sysdatabases WITH(NOLOCK)
WHERE EXISTS(SELECT * FROM master.dbo.sysprocesses WITH(NOLOCK)
             WHERE SPID = @@SPID
             AND sysprocesses.dbid = sysdatabases.dbid);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[061]
IF @dbname = 'odsdb'
BEGIN
     SELECT DISTINCT u.username, p.hostname, p.loginame, b.client_net_address,
            a.start_time AS '开始时间', a.session_id, a.request_id, a.blocking_session_id AS '正在阻塞其他会话的会话ID',
            a.[status] AS '状态', a.command AS '命令', DB_NAME(a.database_id) AS '数据库名',
            a.reads AS '物理读次数', a.writes AS '写次数', a.logical_reads AS '逻辑读次数', a.row_count AS '返回结果行数',
            a.wait_type AS '等待资源类型', a.wait_time AS '等待时间', wait_resource AS '等待的资源',
            (SELECT TOP 1 [text] FROM sys.dm_exec_sql_text(a.sql_handle) ) AS 'sql语句'
     FROM sys.[dm_exec_requests] AS a WITH(NOLOCK)
     INNER JOIN sys.dm_exec_connections AS b WITH(NOLOCK)
     ON a.session_id = b.session_id
     LEFT OUTER JOIN master.dbo.sysprocesses AS p WITH(NOLOCK)
     ON b.session_id = p.spid
     LEFT OUTER JOIN odsdb.dbo.user_dept_detail AS u WITH(NOLOCK)
     ON p.loginame = u.userid
     WHERE a.session_id > 50
     ORDER BY [logical_reads] DESC;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[161]
IF @dbname = 'odsdb_basic'
BEGIN
     SELECT DISTINCT CASE WHEN u.username IS NULL THEN z.LoginName ELSE u.username END AS username,
            p.hostname, p.loginame, b.client_net_address,
            a.start_time AS '开始时间', a.session_id, a.request_id, a.blocking_session_id AS '正在阻塞其他会话的会话ID',
            a.[status] AS '状态', a.command AS '命令', DB_NAME(a.database_id) AS '数据库名',
            a.reads AS '物理读次数', a.writes AS '写次数', a.logical_reads AS '逻辑读次数', a.row_count AS '返回结果行数',
            a.wait_type AS '等待资源类型', a.wait_time AS '等待时间', wait_resource AS '等待的资源',
            (SELECT TOP 1 [text] FROM sys.dm_exec_sql_text(a.sql_handle) ) AS 'sql语句'
     FROM sys.[dm_exec_requests] AS a WITH(NOLOCK)
     INNER JOIN sys.dm_exec_connections AS b WITH(NOLOCK)
     ON a.session_id = b.session_id
     LEFT OUTER JOIN master.dbo.sysprocesses AS p WITH(NOLOCK)
     ON b.session_id = p.spid
     LEFT OUTER JOIN odsdb_basic.dbo.user_dept_detail AS u WITH(NOLOCK)
     ON (CASE WHEN CHARINDEX('_', p.loginame, 1) > 0 THEN LEFT(p.loginame, LEN(p.loginame) - 3) ELSE p.loginame END) = u.userid
     LEFT OUTER JOIN odsdb_basic.dbo.xhq_bi_ip_address AS z WITH(NOLOCK)
     ON b.client_net_address = z.IpAddress
     WHERE a.session_id > 50
     ORDER BY [logical_reads] DESC;
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[备用脚本_1]
SELECT a.scheduler_id, a.[status], a.session_id, a.blocking_session_id, a.reads, a.writes, a.logical_reads,
       a.start_time, a.cpu_time, a.total_elapsed_time AS '执行总时间', d.[name] AS dbname,
       SUBSTRING(LTRIM(b.[text]), a.statement_start_offset / 2 + 1,
                 (CASE WHEN a.statement_end_offset = -1 THEN LEN(CONVERT(NVARCHAR(MAX), b.[text])) * 2 ELSE a.statement_end_offset END
                  - a.statement_start_offset) / 2) AS [正在执行代码],
       b.[text] AS [脚本全部代码]
FROM sys.dm_exec_requests AS a WITH(NOLOCK)
CROSS APPLY sys.dm_exec_sql_text(sql_handle) AS b
LEFT OUTER JOIN sys.databases AS d WITH(NOLOCK)
ON a.database_id = d.database_id
WHERE a.session_id > 50
AND a.session_id <> @@SPID
ORDER BY a.total_elapsed_time DESC;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[备用脚本_2]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @LOGIN_NAME_BAK NVARCHAR(MAX);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @LOGIN_NAME_BAK = N'103922';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[EXEC]
SELECT c.DATABASENAME,
       a.SESSION_ID,
       a.[HOST_NAME],
       a.[PROGRAM_NAME],
       a.CLIENT_INTERFACE_NAME,
       a.LOGIN_NAME,
       a.LOGIN_TIME,
       a.NT_DOMAIN,
       a.NT_USER_NAME,
       b.CLIENT_NET_ADDRESS,
       b.LOCAL_NET_ADDRESS,
       c.OBJNAME,
       c.QUERY
FROM SYS.DM_EXEC_SESSIONS AS a
INNER JOIN SYS.DM_EXEC_CONNECTIONS AS b
ON b.SESSION_ID = a.SESSION_ID
CROSS APPLY(SELECT DB_NAME(DBID) AS DATABASENAME,
                   OBJECT_ID(OBJECTID) AS OBJNAME,
                   ISNULL((SELECT TEXT AS [processing-instruction(definition)]
                           FROM SYS.DM_EXEC_SQL_TEXT(b.MOST_RECENT_SQL_HANDLE)
                           FOR XML PATH(''), TYPE), '') AS QUERY
            FROM SYS.DM_EXEC_SQL_TEXT(b.MOST_RECENT_SQL_HANDLE)) c
WHERE a.SESSION_ID <> @@SPID
AND   a.LOGIN_NAME = @LOGIN_NAME_BAK;
SET NOEXEC OFF;
```

#### 老版本——20220720
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[24小时内执行过的脚本]
GOTO MRX_SKIP;
SELECT a.last_execution_time AS TimeExecute, b.[text] AS Script
FROM SYS.DM_EXEC_QUERY_STATS AS a WITH(NOLOCK)
CROSS APPLY SYS.DM_EXEC_SQL_TEXT(a.sql_handle) AS b
ORDER BY a.last_execution_time DESC;
MRX_SKIP:
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[详细查询]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[当前使用库名]
/*EXEC zhh_useractive--查询运行中的脚本
EXEC zhh_system_sql--查询运行中的脚本*/
DECLARE @dbname VARCHAR(200);
SELECT @dbname = [name]
FROM master.dbo.sysdatabases WITH(NOLOCK)
WHERE EXISTS (SELECT * FROM master.dbo.sysprocesses WITH(NOLOCK)
              WHERE spid = @@SPID AND sysprocesses.dbid = sysdatabases.dbid);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[061]
IF @dbname = 'odsdb'
BEGIN
     SELECT DISTINCT u.username, p.hostname, p.loginame, b.client_net_address,
            a.start_time AS '开始时间', a.session_id, a.request_id, a.blocking_session_id AS '正在阻塞其他会话的会话ID',
            a.[status] AS '状态', a.command AS '命令', DB_NAME(a.database_id) AS '数据库名',
            a.reads AS '物理读次数', a.writes AS '写次数', a.logical_reads AS '逻辑读次数', a.row_count AS '返回结果行数',
            a.wait_type AS '等待资源类型', a.wait_time AS '等待时间', wait_resource AS '等待的资源',
            (SELECT TOP 1 [text] FROM sys.dm_exec_sql_text(a.sql_handle) ) AS 'sql语句'
     FROM sys.[dm_exec_requests] AS a WITH(NOLOCK)
     INNER JOIN sys.dm_exec_connections AS b WITH(NOLOCK)
     ON a.session_id = b.session_id
     LEFT OUTER JOIN master.dbo.sysprocesses AS p WITH(NOLOCK)
     ON b.session_id = p.spid
     LEFT OUTER JOIN odsdb.dbo.user_dept_detail AS u WITH(NOLOCK)
     ON p.loginame = u.userid
     WHERE a.session_id > 50
     ORDER BY [logical_reads] DESC;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[161]
IF @dbname = 'odsdb_basic'
BEGIN
     SELECT DISTINCT CASE WHEN u.username IS NULL THEN z.LoginName ELSE u.username END AS username,
            p.hostname, p.loginame, b.client_net_address,
            a.start_time AS '开始时间', a.session_id, a.request_id, a.blocking_session_id AS '正在阻塞其他会话的会话ID',
            a.[status] AS '状态', a.command AS '命令', DB_NAME(a.database_id) AS '数据库名',
            a.reads AS '物理读次数', a.writes AS '写次数', a.logical_reads AS '逻辑读次数', a.row_count AS '返回结果行数',
            a.wait_type AS '等待资源类型', a.wait_time AS '等待时间', wait_resource AS '等待的资源',
            (SELECT TOP 1 [text] FROM sys.dm_exec_sql_text(a.sql_handle) ) AS 'sql语句'
     FROM sys.[dm_exec_requests] AS a WITH(NOLOCK)
     INNER JOIN sys.dm_exec_connections AS b WITH(NOLOCK)
     ON a.session_id = b.session_id
     LEFT OUTER JOIN master.dbo.sysprocesses AS p WITH(NOLOCK)
     ON b.session_id = p.spid
     LEFT OUTER JOIN odsdb_basic.dbo.user_dept_detail AS u WITH(NOLOCK)
     ON (CASE WHEN CHARINDEX('_', p.loginame, 1) > 0 THEN LEFT(p.loginame, LEN(p.loginame) - 3) ELSE p.loginame END) = u.userid
     LEFT OUTER JOIN odsdb_basic.dbo.xhq_bi_ip_address AS z WITH(NOLOCK)
     ON b.client_net_address = z.IpAddress
     WHERE a.session_id > 50
     ORDER BY [logical_reads] DESC;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[备用脚本]
SELECT a.scheduler_id, a.[status], a.session_id, a.blocking_session_id, a.reads, a.writes, a.logical_reads,
       a.start_time, a.cpu_time, a.total_elapsed_time AS '执行总时间', d.[name] AS dbname,
       SUBSTRING(LTRIM(b.[text]), a.statement_start_offset / 2 + 1,
                 (CASE WHEN a.statement_end_offset = -1 THEN LEN(CONVERT(NVARCHAR(MAX), b.[text])) * 2 ELSE a.statement_end_offset END
                  - a.statement_start_offset) / 2) AS [正在执行代码],
       b.[text] AS [脚本全部代码]
FROM sys.dm_exec_requests AS a WITH(NOLOCK)
CROSS APPLY sys.dm_exec_sql_text(sql_handle) AS b
LEFT OUTER JOIN sys.databases AS d WITH(NOLOCK)
ON a.database_id = d.database_id
WHERE a.session_id > 50
AND a.session_id <> @@SPID
ORDER BY a.total_elapsed_time DESC;
```

<hr style="border:2px solid DeepSkyBlue"> </hr>

### 在跑作业——MSYS_job_running
#### 方案一—20220402
```SQL
EXEC msdb.dbo.SP_HELP_JOB @EXECUTION_STATUS = 1;
```

#### 方案二—20220402
```SQL
SET NOEXEC ON;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CREATE TABLE]
IF OBJECT_ID('tempdb.dbo.#RunningJobs', 'u') IS NOT NULL DROP TABLE #RunningJobs;
CREATE TABLE #RunningJobs(Job_ID UNIQUEIDENTIFIER,
                          Last_Run_Date INT,
                          Last_Run_Time INT,
                          Next_Run_Date INT,
                          Next_Run_Time INT,
                          Next_Run_Schedule_ID INT,
                          Requested_To_Run INT,
                          Request_Source INT,
                          Request_Source_ID VARCHAR(100),
                          Running INT,
                          Current_Step INT,
                          Current_Retry_Attempt INT,
                          STATE INT);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[xp_sqlagent_enum_jobs]
INSERT INTO #RunningJobs
EXEC MASTER.dbo.xp_sqlagent_enum_jobs 1, garbage;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[RESULT]
SELECT b.name AS JobName,
       CASE WHEN Next_Run_Date = 0
            THEN 'Not scheduled'
            ELSE CONVERT(VARCHAR, DATEADD(S, (Next_Run_Time / 10000) * 60 * 60/*hours*/
                                          + ((Next_Run_Time - (Next_Run_Time / 10000) * 10000) / 100) * 60/*mins*/
                                          + (Next_Run_Time - (Next_Run_Time / 100) * 100),/*secs*/
                                          CONVERT(DATETIME, RTRIM(Next_Run_Date), 112)), 100) END AS StartTime
FROM #RunningJobs AS a WITH(NOLOCK)
INNER JOIN msdb.dbo.sysjobs AS b WITH(NOLOCK)
ON a.JOB_ID = b.JOB_ID
WHERE Running = 1
ORDER BY Next_Run_Date DESC, Next_Run_Time DESC;
SET NOEXEC OFF;
```

#### 方案三—20220402
```SQL
SELECT a.name,
a.job_id,
a.originating_server,
b.run_requested_date,
DATEDIFF(SECOND, b.run_requested_date, GETDATE()) AS Elapsed
FROM msdb.dbo.sysjobs_view AS a WITH(NOLOCK)
INNER JOIN msdb.dbo.sysjobactivity AS b WITH(NOLOCK)
ON a.JOB_ID = b.JOB_ID
INNER JOIN msdb.dbo.syssessions AS c WITH(NOLOCK)
ON c.SESSION_ID = b.SESSION_ID
INNER JOIN(SELECT MAX(agent_start_date) AS max_agent_start_date
           FROM msdb.dbo.syssessions WITH(NOLOCK)) AS d
ON c.agent_start_date = d.max_agent_start_date
WHERE run_requested_date IS NOT NULL
AND   stop_execution_date IS NULL;
```

<hr style="border:2px solid DeepSkyBlue"> </hr>

### 枪毙查询——MSYS_KILL
#### 新版本—20220307
```SQL
/*[1]KILL LIST*/SELECT TOP 300 * FROM dbo.KILL_RUNNINGJOB WITH(NOLOCK) ORDER BY DEALTIME DESC;
/*[2]枪毙明细关联老大哥监控明细*/
DECLARE @HOUR_AGO INT;
SET @HOUR_AGO = 24;
SELECT TOP 300 * 
FROM dbo.KILL_RUNNINGJOB AS a WITH(NOLOCK)/*枪毙明细*/
INNER JOIN XTB_BIGBROTHER AS b WITH(NOLOCK)/*老大哥监控明细*/
ON a.开始时间 = b.START_TIME
AND a.SESSION_ID = b.SESSION_ID
WHERE b.COLLECTION_TIME >= DATEADD(MINUTE, -5, a.DEALTIME)
AND   a.DEALTIME >= DATEADD(HOUR, -1 * @HOUR_AGO, CURRENT_TIMESTAMP)
ORDER BY a.DEALTIME DESC, a.session_id DESC
/*[3]NOT KILL WHITELIST*/SELECT * FROM SessionID_WithoutKill WITH(NOLOCK);
/*[4]增加防止作弊作业目录*/SELECT * FROM CHEAT_JOB_ID WITH(NOLOCK);
/*[5]老大哥监控明细*/
EXEC [PS_BigBrother];
/*[5.1]最新*/SELECT * FROM XTB_BIGBROTHER_LAST WITH(NOLOCK) ORDER BY TEMPDB_ALLOCATIONS + TEMPDB_CURRENT DESC;
/*[5.2]特定*/
DECLARE @SESSION_ID INT;
                                                                                        /*#TEMPORARY#*/SET @SESSION_ID = 252;
WITH CTE_ALLTIME
AS (SELECT MAX(start_time) AS start_time,
           MAX([dd hh:mm:ss.mss]) AS [dd hh:mm:ss.mss]
    FROM XTB_BigBrother WITH(NOLOCK)
    WHERE start_time >= DATEADD(HOUR, -24, CURRENT_TIMESTAMP)/*过去24小时*/
      AND SESSION_ID = @SESSION_ID)
SELECT a.*
FROM XTB_BigBrother    AS a WITH(NOLOCK)
INNER JOIN CTE_ALLTIME AS b WITH(NOLOCK)
ON a.start_time = b.start_time
AND a.[dd hh:mm:ss.mss] = b.[dd hh:mm:ss.mss]
WHERE a.start_time >= DATEADD(HOUR, -24, CURRENT_TIMESTAMP);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[1-综合查询服务器]
WITH CTE_ALL_SERVER
AS (SELECT TOP 300
           '061' AS SERVERNAME, JOB_NAME, STYLE, USERNAME, HOSTNAME, 开始时间, SESSION_ID, SQL执行语句, DEALTIME, 等待资源类型, 运行时间, KILL_STYLE
    FROM dbo.KILL_RUNNINGJOB WITH(NOLOCK)
    ORDER BY DEALTIME DESC
    UNION ALL
    SELECT TOP 300
           '161' AS SERVERNAME, JOB_NAME, STYLE, USERNAME, HOSTNAME, 开始时间, SESSION_ID, SQL执行语句, DEALTIME, 等待资源类型, 运行时间, KILL_STYLE
    FROM DATABASE_5_161.odsdb_basic.dbo.KILL_RUNNINGJOB WITH(NOLOCK)
    ORDER BY DEALTIME DESC
    UNION ALL
    SELECT TOP 300
           '162' AS SERVERNAME, JOB_NAME, STYLE, USERNAME, HOSTNAME, 开始时间, SESSION_ID, SQL执行语句, DEALTIME, 等待资源类型, 运行时间, KILL_STYLE
    FROM DATABASE_5_162.odsdb_basic.dbo.KILL_RUNNINGJOB WITH(NOLOCK)
    ORDER BY DEALTIME DESC
    UNION ALL
    SELECT TOP 300
           '028' AS SERVERNAME, JOB_NAME, STYLE, USERNAME, HOSTNAME, 开始时间, SESSION_ID, SQL执行语句, DEALTIME, 等待资源类型, 运行时间, KILL_STYLE
    FROM DATABASE_28_HN.odsdbfq.dbo.KILL_RUNNINGJOB WITH(NOLOCK)
    ORDER BY DEALTIME DESC
    UNION ALL
    SELECT TOP 300
           '029' AS SERVERNAME, JOB_NAME, STYLE, USERNAME, HOSTNAME, 开始时间, SESSION_ID, SQL执行语句, DEALTIME, 等待资源类型, 运行时间, KILL_STYLE
    FROM DATABASE_28_HN.odsdbfq.dbo.VIEW_KILL_RUNNINGJOB_FROM_2_29 WITH(NOLOCK)
    ORDER BY DEALTIME DESC),
    DEALTIME_APART_DAY
AS (SELECT CONVERT(CHAR(8), DEALTIME, 112) AS DAYTIME,
    SERVERNAME, JOB_NAME, STYLE, USERNAME, HOSTNAME, 开始时间, SESSION_ID, SQL执行语句, DEALTIME, 等待资源类型, 运行时间, KILL_STYLE
    FROM CTE_ALL_SERVER WITH(NOLOCK))
SELECT ROW_NUMBER()OVER(PARTITION BY DAYTIME, SERVERNAME ORDER BY DEALTIME DESC) AS ROWGROUP,
       SERVERNAME, JOB_NAME, STYLE, USERNAME, HOSTNAME, 开始时间, SESSION_ID, SQL执行语句, DEALTIME, 等待资源类型, 运行时间, KILL_STYLE
FROM DEALTIME_APART_DAY WITH(NOLOCK)
ORDER BY DAYTIME DESC, SERVERNAME ASC;
```

#### 老版本—20220301
```SQL
/*[1]KILL LIST*/SELECT TOP 300 * FROM dbo.KILL_RUNNINGJOB WITH(NOLOCK) ORDER BY DEALTIME DESC;
/*[2]枪毙明细关联老大哥监控明细*/
DECLARE @HOUR_AGO INT;
SET @HOUR_AGO = 24;
SELECT TOP 300 * 
FROM dbo.KILL_RUNNINGJOB AS a WITH(NOLOCK)/*枪毙明细*/
INNER JOIN XTB_BIGBROTHER AS b WITH(NOLOCK)/*老大哥监控明细*/
ON a.开始时间 = b.START_TIME
AND a.SESSION_ID = b.SESSION_ID
WHERE b.COLLECTION_TIME >= DATEADD(MINUTE, -5, a.DEALTIME)
AND   a.DEALTIME >= DATEADD(HOUR, -1 * @HOUR_AGO, CURRENT_TIMESTAMP)
ORDER BY a.DEALTIME DESC, a.session_id DESC
/*[3]NOT KILL WHITELIST*/SELECT * FROM SessionID_WithoutKill WITH(NOLOCK);
/*[4]增加防止作弊作业目录*/SELECT * FROM CHEAT_JOB_ID WITH(NOLOCK);
/*[5]老大哥监控明细*/
EXEC [PS_BigBrother];
/*[5.1]最新*/SELECT * FROM XTB_BIGBROTHER_LAST WITH(NOLOCK) ORDER BY TEMPDB_ALLOCATIONS + TEMPDB_CURRENT DESC;
/*[5.2]特定*/
DECLARE @SESSION_ID INT;
                                                                                        /*#TEMPORARY#*/SET @SESSION_ID = 252;
WITH CTE_ALLTIME
AS (SELECT MAX(start_time) AS start_time,
           MAX([dd hh:mm:ss.mss]) AS [dd hh:mm:ss.mss]
    FROM XTB_BigBrother WITH(NOLOCK)
    WHERE start_time >= DATEADD(HOUR, -24, CURRENT_TIMESTAMP)/*过去24小时*/
      AND SESSION_ID = @SESSION_ID)
SELECT a.*
FROM XTB_BigBrother    AS a WITH(NOLOCK)
INNER JOIN CTE_ALLTIME AS b WITH(NOLOCK)
ON a.start_time = b.start_time
AND a.[dd hh:mm:ss.mss] = b.[dd hh:mm:ss.mss]
WHERE a.start_time >= DATEADD(HOUR, -24, CURRENT_TIMESTAMP);
```

<hr style="border:2px solid DeepSkyBlue"> </hr>

### 性能监控——MSYS_WEB
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[监控页面]
--061--http://zabbix.myj.com.cn/zabbix.php?action=charts.view&view_as=showgraph&filter_hostids%5B%5D=10167&filter_set=1
--161--http://zabbix.myj.com.cn/zabbix.php?action=charts.view&view_as=showgraph&filter_hostids%5B%5D=10537&filter_set=1
--162--http://zabbix.myj.com.cn/zabbix.php?action=charts.view&view_as=showgraph&filter_hostids%5B%5D=10538&filter_set=1
--167--http://zabbix.myj.com.cn/zabbix.php?action=charts.view&view_as=showgraph&filter_hostids%5B%5D=10575&filter_set=1
--064--http://zabbix.myj.com.cn/zabbix.php?action=charts.view&view_as=showgraph&filter_hostids%5B%5D=10923&filter_set=1
--065--http://zabbix.myj.com.cn/zabbix.php?action=charts.view&view_as=showgraph&filter_hostids%5B%5D=10924&filter_set=1
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SQL MONITOR]
--账号: Read-only user
--密码: Read@2021
--http://sql.myj.lan/overviews/192.168.5.37/cluster/192.168.0.61/sql/(local)#?MaxTime=1660546210121&Alert=1025547&Zoom=1660508836893%2C1660516079000
```

<hr style="border:2px solid DeepSkyBlue"> </hr>

### 性能监控——MSYS_PS
```SQL
--PS_CPU-监控服务器概况↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨[[1\|1]]
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[S1]
--如果返回的是一条记录，说明服务器不支持NUMA架构，否则记录数就是NUMA架构的节点数(NUMA：非均匀访存模型)
--Node_id：NUMA节点id
--No.of CPU in the NUMA：分配给NUMA节点的CPU数或调度数(number of schedulers)
--Total No. of CPU：服务器上可用CPU总数
--Runnable Task Count：在可运行队列里，等待被重现调度的，用于分配任务(tasks)的工作者数(workers)。即可运行队列里请求数。
--Pending disk I/O count：等待被完成的等待IO数。每个调度都有一个等待IO清单，用于判断它们在上下文切换时是否完成。
--                        当请求被插入时，这个数字会增加。请求完成后，数字会减少。
--Work queue count：等待队列里的任务数。这些任务等待工作者拿走。
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[注释版]
SET NOEXEC ON;
SELECT PARENT_NODE_ID AS NODE_ID,/*NUMA节点ID*/
       COUNT(*) AS [No.of CPU In the NUMA],/*CPU数或调度数*/
       SUM(COUNT(*))OVER() AS [Total No. of CPU],/*可用CPU总数*/
       SUM(runnable_tasks_count) AS [Runnable Task Count],/*用于分配任务(tasks)的工作者数(workers)*/
       SUM(pending_disk_io_count) AS [Pending disk I/O count],/*等待被完成的等待IO数*/
       SUM(work_queue_count) AS [Work queue count]/*等待队列里的任务数*/
FROM sys.dm_os_schedulers
WHERE status = 'VISIBLE ONLINE'
GROUP BY PARENT_NODE_ID;
SET NOEXEC OFF;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[运行版]
SELECT PARENT_NODE_ID AS NODE_ID,
       COUNT(*) AS [No.of CPU In the NUMA],
       SUM(COUNT(*))OVER() AS [Total No. of CPU],
       SUM(runnable_tasks_count) AS [Runnable Task Count],
       SUM(pending_disk_io_count) AS [Pending disk I/O count],
       SUM(work_queue_count) AS [Work queue count]
FROM sys.dm_os_schedulers
WHERE status = 'VISIBLE ONLINE'
GROUP BY PARENT_NODE_ID;
--PS_WAIT-监控进程等待时间↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨[[2\|2]]
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[S1:PREREQUISITE FUNCTION]
SET NOEXEC ON;
USE MASTER;
GO
CREATE FUNCTION ConvertStringToBinary(@HEXSTRING VARCHAR(100))
RETURNS BINARY(34)
AS
BEGIN
     RETURN(SELECT CAST('' AS XML).value('xs:hexBinary( substring(sql:variable("@hexstring"), sql:column("t.pos")) )', 'varbinary(max)')
            FROM(SELECT CASE SUBSTRING(@HEXSTRING, 1, 2)
                        WHEN '0x' THEN 3
                        ELSE 0 END) AS t(POS))
END
SET NOEXEC OFF;
--PS_WAIT↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[S2:LIST the SESSION which ARE currently waiting FOR RESOURCE]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[注释版]
SET NOEXEC ON;
WITH CTE_DM_OS_WAITING_TASKS
AS (SELECT SESSION_ID,
    SUM(WAIT_DURATION_MS) AS WAIT_DURATION_MS,
    WAIT_TYPE,
    BLOCKING_SESSION_ID,
    COUNT(*) AS NoThread
    FROM sys.dm_os_waiting_tasks
    GROUP BY SESSION_ID, WAIT_TYPE, BLOCKING_SESSION_ID),
CTE_SCHEDULERS_WORKERS
AS (SELECT OS.parent_node_id,
           OSW.task_address
           FROM sys.dm_os_schedulers    OS
           INNER JOIN sys.dm_os_workers OSW
           ON OS.scheduler_address = OSW.scheduler_address
           WHERE OS.status = 'VISIBLE ONLINE'
           GROUP BY OS.parent_node_id, task_address)
SELECT node.parent_node_id AS NODE_ID,/*Node_id——可以被调度者查询的节点映射*/
       c.host_name,/*建立连接的计算机名*/
       c.login_name,/*会话用户名*/
       CASE WHEN c.program_name LIKE '%SQLAgent - TSQL JobStep%'
            THEN (SELECT 'SQL AGENT JOB: ' + name
                  FROM msdb.dbo.sysjobs
                  WHERE job_id = MASTER.dbo.ConvertStringToBinary
                                 (LTRIM(RTRIM((SUBSTRING(c.program_name, CHARINDEX('(job', c.program_name, 0) + 4, 35))))))
            ELSE c.program_name
            END AS [Program Name],/*使用会话的对应程序名|SQL Server代理显示作业名*/
       DB_NAME(b.DATABASE_ID) AS DatabaseName,/*会话的当前数据库名*/
       b.SESSION_ID,/*会话ID*/
       a.blocking_session_id,/*阻塞语句的会话ID*/
       a.wait_duration_ms,/*等待时间单位为毫秒★★★★★*/
       a.wait_type,/*等待类型名称*/
       a.NoThread,/*当前会话的线程数*/
       b.command,/*标识当前类型的命令*/
       b.status,/*请求状态：Background,Running,Runnable,Sleeping,Suspended*/
       b.wait_resource,/*请求当前等待的资源*/
       b.open_transaction_count,/*当前会话打开的事务数*/
       b.cpu_time,/*请求使用的CPU时间，单位毫秒*/
       b.total_elapsed_time AS ElapsedTime_ms,/*自请求到达后占用的CPU时间，单位毫秒★★★★★*/
       b.percent_complete,/*指定操作的工作完成进度，例如备份、还原、回滚等*/
       b.reads,/*reads——请求执行的读数*/
       b.writes,/*writes——请求执行的写数*/
       b.logical_reads,/*logical_reads——请求执行的逻辑读数*/
       d.name AS ResoursePool,/*资源管理池名称*/
       SUBSTRING(sqltxt.text, (b.statement_start_offset / 2) + 1,
                 ((CASE WHEN b.statement_end_offset = -1 THEN LEN(CONVERT(NVARCHAR(MAX), sqltxt.text)) * 2 ELSE b.statement_end_offset END
                   - b.statement_start_offset) / 2) + 1) AS [Individual Query],/*Individual Query——在会话里运行的批处理SQL语句*/
       sqltxt.text AS [Batch Query]/*Batch Query——在会话里运行的批处理(存储过程/一系列的语句)*/
FROM CTE_DM_OS_WAITING_TASKS AS a WITH(NOLOCK)
INNER JOIN sys.dm_exec_requests b WITH(NOLOCK)
ON a.SESSION_ID = b.SESSION_ID
INNER JOIN sys.dm_exec_sessions c WITH(NOLOCK)
ON c.SESSION_ID = b.SESSION_ID
INNER JOIN sys.dm_resource_governor_workload_groups d WITH(NOLOCK)
ON d.GROUP_ID = b.GROUP_ID
INNER JOIN CTE_SCHEDULERS_WORKERS AS node WITH(NOLOCK)
ON node.TASK_ADDRESS = b.TASK_ADDRESS
CROSS APPLY sys.dm_exec_sql_text(b.sql_handle) AS sqltxt
WHERE b.sql_handle IS NOT NULL
AND a.WAIT_TYPE NOT IN ('WAITFOR', 'BROKER_RECEIVE_WAITFOR');
SET NOEXEC OFF;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[运行版]
WITH CTE_DM_OS_WAITING_TASKS
AS (SELECT SESSION_ID,
    SUM(WAIT_DURATION_MS) AS WAIT_DURATION_MS,
    WAIT_TYPE,
    BLOCKING_SESSION_ID,
    COUNT(*) AS NoThread
    FROM sys.dm_os_waiting_tasks
    GROUP BY SESSION_ID, WAIT_TYPE, BLOCKING_SESSION_ID),
CTE_SCHEDULERS_WORKERS
AS (SELECT OS.parent_node_id,
           OSW.task_address
           FROM sys.dm_os_schedulers    OS
           INNER JOIN sys.dm_os_workers OSW
           ON OS.scheduler_address = OSW.scheduler_address
           WHERE OS.status = 'VISIBLE ONLINE'
           GROUP BY OS.parent_node_id, task_address)
SELECT node.parent_node_id AS NODE_ID,
       c.host_name,
       c.login_name,
       CASE WHEN c.program_name LIKE '%SQLAgent - TSQL JobStep%'
            THEN (SELECT 'SQL AGENT JOB: ' + name
                  FROM msdb.dbo.sysjobs
                  WHERE job_id = MASTER.dbo.ConvertStringToBinary
                                 (LTRIM(RTRIM((SUBSTRING(c.program_name, CHARINDEX('(job', c.program_name, 0) + 4, 35))))))
            ELSE c.program_name
            END AS [Program Name],
       DB_NAME(b.DATABASE_ID) AS DatabaseName,
       b.SESSION_ID,
       a.blocking_session_id,
       a.wait_duration_ms,/*FOCUS*/
       a.wait_type,
       a.NoThread,
       b.command,
       b.status,
       b.wait_resource,
       b.open_transaction_count,
       b.cpu_time,
       b.total_elapsed_time AS ElapsedTime_ms,/*FOCUS*/
       b.percent_complete,
       b.reads,
       b.writes,
       b.logical_reads,
       d.name AS ResoursePool,
       SUBSTRING(sqltxt.text, (b.statement_start_offset / 2) + 1,
                 ((CASE WHEN b.statement_end_offset = -1 THEN LEN(CONVERT(NVARCHAR(MAX), sqltxt.text)) * 2 ELSE b.statement_end_offset END
                   - b.statement_start_offset) / 2) + 1) AS [Individual Query],
       sqltxt.text AS [Batch Query]
FROM CTE_DM_OS_WAITING_TASKS AS a WITH(NOLOCK)
INNER JOIN sys.dm_exec_requests b WITH(NOLOCK)
ON a.SESSION_ID = b.SESSION_ID
INNER JOIN sys.dm_exec_sessions c WITH(NOLOCK)
ON c.SESSION_ID = b.SESSION_ID
INNER JOIN sys.dm_resource_governor_workload_groups d WITH(NOLOCK)
ON d.GROUP_ID = b.GROUP_ID
INNER JOIN CTE_SCHEDULERS_WORKERS AS node WITH(NOLOCK)
ON node.TASK_ADDRESS = b.TASK_ADDRESS
CROSS APPLY sys.dm_exec_sql_text(b.sql_handle) AS sqltxt
WHERE b.sql_handle IS NOT NULL
AND a.WAIT_TYPE NOT IN ('WAITFOR', 'BROKER_RECEIVE_WAITFOR');
--PS_BLOCK-监控堵塞进程↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨[[3\|3]]
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[S1:List the current blocking session information]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[FUNCTION]
SET NOEXEC ON;
USE MASTER;
GO
CREATE FUNCTION [dbo].GetStatementForSpid(@SPID SMALLINT)
RETURNS NVARCHAR(4000)
BEGIN
     /*DECLARE*/
     DECLARE @SqlHandle BINARY(20);
     DECLARE @SQLTEXT NVARCHAR(MAX);
     /*SET*/
     SELECT @SQLHANDLE = SQL_HANDLE
     FROM sys.sysprocesses WITH(NOLOCK)
     WHERE SPID = @SPID;
     SELECT @SQLTEXT = [TEXT]
     FROM sys.dm_exec_sql_text(@SQLHANDLE);
     /*RETURN*/
     RETURN @SQLTEXT;
END
GO
SET NOEXEC OFF;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶
SELECT b.SESSION_ID,
       b.HOST_NAME,
       DB_NAME(a.DATABASE_ID)  AS DatabaseName,
       CASE WHEN b.program_name LIKE '%SQLAgent - TSQL JobStep%'
            THEN (SELECT 'SQL AGENT JOB: ' + name
                  FROM msdb.dbo.sysjobs
                  WHERE job_id = MASTER.dbo.ConvertStringToBinary(LTRIM(RTRIM(
                                 (SUBSTRING(b.program_name, CHARINDEX('(job', b.program_name, 0) + 4, 35))
                                  ))))
            ELSE b.program_name
       END AS program_name,
       b.login_name,
       master.dbo.GetStatementForSpid(b.session_id) AS [Statement],
       c.session_id AS Blocking_Session_ID,
       c.host_name AS Blocking_Hostname,
       CASE WHEN c.program_name LIKE '%SQLAgent - TSQL JobStep%'
            THEN (SELECT 'SQL AGENT JOB: ' + name
                  FROM msdb.dbo.sysjobs
                  WHERE job_id = MASTER.dbo.ConvertStringToBinary(LTRIM(RTRIM(
                                 (SUBSTRING(c.program_name, CHARINDEX('(job', b.program_name, 0) + 4, 35))
                                  ))))
            ELSE c.program_name
       END AS Blocking_program_name,
       c.login_name AS Blocking_login_name,
       MASTER.dbo.GetStatementForSpid(c.session_id) AS Blocking_Statement
FROM sys.dm_exec_requests a WITH(NOLOCK)
INNER JOIN sys.dm_exec_sessions b WITH(NOLOCK)
ON b.SESSION_ID = a.SESSION_ID
INNER JOIN sys.dm_exec_sessions c WITH(NOLOCK)
ON c.SESSION_ID = a.BLOCKING_SESSION_ID;
--PS_NOT_ACTIVE-监控读写停滞进程↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[S2:List the Open session with transaction which is not active]
--SELECT * FROM sys.dm_exec_sessions WHERE status = 'running' ORDER BY login_name;
--SELECT * FROM sys.dm_tran_session_transactions;
--SELECT * FROM sys.dm_exec_connections WHERE parent_connection_id IS NULL ORDER BY session_id;
--SELECT * FROM sys.sysprocesses WHERE ecid = 0 ORDER BY spid;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @TIMELENGTH_MINUTE INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @TIMELENGTH_MINUTE = 2
--SET @TIMELENGTH_MINUTE = @TIMELENGTH_MINUTE_INPUT/*传参分钟数*/
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[结果写入]
--IF OBJECT_ID('.dbo.XTB_NOT_ACTIVE', 'U') IS NOT NULL DROP TABLE dbo.XTB_NOT_ACTIVE;
SELECT a.SESSION_ID,
       a.LOGIN_NAME,
       a.HOST_NAME,
       CASE WHEN a.PROGRAM_NAME LIKE '%SQLAgent - TSQL JobStep%'
            THEN(SELECT 'SQL AGENT JOB: ' + name
                 FROM msdb.dbo.sysjobs
                 WHERE JOB_ID = master.dbo.ConvertStringToBinary
                                (LTRIM(RTRIM((SUBSTRING(a.PROGRAM_NAME, CHARINDEX('(job', a.PROGRAM_NAME, 0) + 4, 35))))))
            ELSE a.PROGRAM_NAME
       END AS PROGRAM_NAME,
       DB_NAME(d.DBID) AS DatabaseName,
       d.LASTWAITTYPE,
       f.text,
       c.last_read,
       c.last_write
--INTO dbo.XTB_NOT_ACTIVE
FROM sys.dm_exec_sessions AS a
INNER JOIN sys.dm_tran_session_transactions AS b
ON a.SESSION_ID = b.SESSION_ID
INNER JOIN sys.dm_exec_connections AS c
ON a.SESSION_ID = c.SESSION_ID
INNER JOIN sys.sysprocesses AS d
ON d.SPID = a.SESSION_ID
LEFT OUTER JOIN sys.dm_exec_requests AS e
ON b.SESSION_ID = e.SESSION_ID AND e.SESSION_ID IS NULL
CROSS APPLY sys.dm_exec_sql_text(c.most_recent_sql_handle) AS f
WHERE a.status = 'running'/*PLUS*/
AND c.parent_connection_id IS NULL/*PLUS*/
AND d.ecid = 0
AND (DATEDIFF(SS, c.LAST_READ, GETDATE()) + DATEDIFF(SS, c.LAST_WRITE, GETDATE())) >= (@TIMELENGTH_MINUTE * 60)
AND d.LASTWAITTYPE NOT IN('BROKER_RECEIVE_WAITFOR', 'WAITFOR');
--PS_HEAVY_QUERY-监控最耗资源进程↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨↨[[4\|4]]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[注释版]
IF OBJECT_ID('.dbo.XTB_PS_HEAVY_QUERY', 'U') IS NOT NULL DROP TABLE dbo.XTB_PS_HEAVY_QUERY;
SELECT ROW_NUMBER()OVER(ORDER BY CAST((a.total_worker_time) / 1000000.0 AS DECIMAL(28, 2)) DESC) AS NUM,
       DB_NAME(b.DBID) AS DatabaseName,
/*DatabaseName——执行计划的数据库环境(数据库名)*/
       DATEDIFF(MI, a.creation_time, GETDATE()) AS [Age of the Plan(Minutes)],
/*Age of the Plan(Minutes)——计划缓存里计划的生存期，单位为分钟*/
       a.last_execution_time AS [Last Execution Time],
/*Last Execution Time——这个计划的上次执行日期和时间*/
       a.execution_count AS [Total Execution Count],
/*Total Execution Count——自上次编译后，总执行次数；在执行计划生存期内[Age of the Plan(Minutes)]，总执行次数(自上次编译后)*/
       CAST((a.total_elapsed_time) / 1000000.0 AS DECIMAL(28, 2)) [Total Elapsed Time(s)],
/*Total Elapsed Time(s)——执行这个计划总执行次数后[Total Execution Count]的总占用时间，单位为秒*/
       CAST((a.total_elapsed_time) / 1000000.0 / a.execution_count AS DECIMAL(28, 2)) AS [Average Execution time(s)],
/*Average Execution time(s)——这个计划每次执行的平均时间，单位为秒*/
       CAST((a.total_worker_time) / 1000000.0 AS DECIMAL(28, 2)) AS [Total CPU time (s)],
/*Total CPU time (s)——执行这个计划总执行次数后[Total Execution Count]的总CPU时间，单位为秒*/
       CAST(a.total_worker_time * 100.0 / a.total_elapsed_time AS DECIMAL(28, 2)) AS [% CPU],
/*% CPU——与Total Elapsed Time(s)相比，CPU占用时间比*/
       CAST((a.total_elapsed_time - a.total_worker_time) * 100.0 / a.total_elapsed_time AS DECIMAL(28, 2)) AS [% Waiting],
/*% Waiting——与Total Elapsed Time(s)相比，等待资源占用时间比*/
       CAST((a.total_worker_time) / 1000000.0 / a.execution_count AS DECIMAL(28, 2)) AS [CPU time average (s)],
/*CPU time average (s)——每次执行的平均CPU时间，单位为秒*/
       CAST((a.total_physical_reads) / a.execution_count AS DECIMAL(28, 2)) AS [Avg Physical Read],
/*Avg Physical Read——每次执行的平均物理读数*/
       CAST((a.total_logical_reads) / a.execution_count AS DECIMAL(28, 2)) AS [Avg Logical Reads],
/*Avg Logical Reads——每次执行的平均逻辑读数*/
       CAST((a.total_logical_writes) / a.execution_count AS DECIMAL(28, 2)) AS [Avg Logical Writes],
/*Avg Logical Writes——每次执行的平均逻辑写数*/
       max_physical_reads,
/*max_physical_reads——每次执行的时候，出新最大物理读数*/
       max_logical_reads,
/*max_logical_reads——每次执行的时候，出新最大逻辑读数*/
       max_logical_writes,
/*max_logical_writes——每次执行的时候，出新最大逻辑写数*/
       SUBSTRING(b.TEXT, (a.statement_start_offset / 2) + 1,
                 ((CASE WHEN a.statement_end_offset = -1
                        THEN LEN(CONVERT(NVARCHAR(MAX), b.text)) * 2
                        ELSE a.statement_end_offset END - a.statement_start_offset) / 2) + 1) AS [Individual Query],
/*Individual Query——批处理语句的部分信息*/
       b.text AS [Batch Statement]--,
/*Batch Statement——批处理查询*/
       --c.query_plan
/*query_plan XML格式的执行计划，点击后我们可以看图示执行计划*/
INTO dbo.XTB_PS_HEAVY_QUERY
FROM sys.dm_exec_query_stats AS a WITH(NOLOCK)
CROSS APPLY sys.dm_exec_sql_text(a.sql_handle) AS b
--CROSS APPLY sys.dm_exec_query_plan(a.plan_handle) AS c
WHERE a.total_elapsed_time > 0
AND a.last_execution_time >= DATEADD(MINUTE, -30, CURRENT_TIMESTAMP)/*PLUS*/
ORDER BY [Total CPU time (s)] DESC;
--[Avg Physical Read]
--[Avg Logical Reads]
--[Avg Logical Writes]
--[Total Elapsed Time(s)]
--[Total Execution Count]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[运行版]
IF OBJECT_ID('.dbo.XTB_PS_HEAVY_QUERY', 'U') IS NOT NULL DROP TABLE dbo.XTB_PS_HEAVY_QUERY;
SELECT ROW_NUMBER()OVER(ORDER BY CAST((a.total_worker_time) / 1000000.0 AS DECIMAL(28, 2)) DESC) AS NUM,
       DB_NAME(b.DBID) AS DatabaseName,
       DATEDIFF(MI, a.creation_time, GETDATE()) AS [Age of the Plan(Minutes)],
       a.last_execution_time AS [Last Execution Time],
       a.execution_count AS [Total Execution Count],
       CAST((a.total_elapsed_time) / 1000000.0 AS DECIMAL(28, 2)) [Total Elapsed Time(s)],
       CAST((a.total_elapsed_time) / 1000000.0 / a.execution_count AS DECIMAL(28, 2)) AS [Average Execution time(s)],
       CAST((a.total_worker_time) / 1000000.0 AS DECIMAL(28, 2)) AS [Total CPU time (s)],
       CAST(a.total_worker_time * 100.0 / a.total_elapsed_time AS DECIMAL(28, 2)) AS [% CPU],
       CAST((a.total_elapsed_time - a.total_worker_time) * 100.0 / a.total_elapsed_time AS DECIMAL(28, 2)) AS [% Waiting],
       CAST((a.total_worker_time) / 1000000.0 / a.execution_count AS DECIMAL(28, 2)) AS [CPU time average (s)],
       CAST((a.total_physical_reads) / a.execution_count AS DECIMAL(28, 2)) AS [Avg Physical Read],
       CAST((a.total_logical_reads) / a.execution_count AS DECIMAL(28, 2)) AS [Avg Logical Reads],
       CAST((a.total_logical_writes) / a.execution_count AS DECIMAL(28, 2)) AS [Avg Logical Writes], max_physical_reads,
       max_logical_reads,
       max_logical_writes,
       SUBSTRING(b.TEXT, (a.statement_start_offset / 2) + 1,
                 ((CASE WHEN a.statement_end_offset = -1
                        THEN LEN(CONVERT(NVARCHAR(MAX), b.text)) * 2
                        ELSE a.statement_end_offset END - a.statement_start_offset) / 2) + 1) AS [Individual Query],
       b.text AS [Batch Statement]--,
       --c.query_plan
INTO dbo.XTB_PS_HEAVY_QUERY
FROM sys.dm_exec_query_stats AS a WITH(NOLOCK)
CROSS APPLY sys.dm_exec_sql_text(a.sql_handle) AS b
--CROSS APPLY sys.dm_exec_query_plan(a.plan_handle) AS c
WHERE a.total_elapsed_time > 0
AND a.last_execution_time >= DATEADD(MINUTE, -30, CURRENT_TIMESTAMP)/*PLUS*/
ORDER BY [Total CPU time (s)] DESC;
--[Avg Physical Read]
--[Avg Logical Reads]
--[Avg Logical Writes]
--[Total Elapsed Time(s)]
--[Total Execution Count]
```

<hr style="border:2px solid DeepSkyBlue"> </hr>

### 资源占用——MSYS_TEMPDB
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[测试结果]
SELECT *
FROM msdb.dbo.sysjobs WITH(NOLOCK)
WHERE JOB_ID =  select dbo.GetJobIdFromProgramName('SQLAgent - TSQL JobStep (Job 0xECC4D10D8CFD94458B0B7640AB35D37A : Step 9)');
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[创建函数]
SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO
CREATE FUNCTION dbo.GetJobIdFromProgramName(@PROGRAM_NAME NVARCHAR(128))
RETURNS UNIQUEIDENTIFIER
AS
BEGIN
     DECLARE @START_OF_JOB_ID INT;
     SET @START_OF_JOB_ID = CHARINDEX('(Job 0x', @PROGRAM_NAME) + 7;
     RETURN CASE WHEN @START_OF_JOB_ID > 0
                 THEN CAST(SUBSTRING(@PROGRAM_NAME, @START_OF_JOB_ID + 06, 2) + 
                           SUBSTRING(@PROGRAM_NAME, @START_OF_JOB_ID + 04, 2) + 
                           SUBSTRING(@PROGRAM_NAME, @START_OF_JOB_ID + 02, 2) + 
                           SUBSTRING(@PROGRAM_NAME, @START_OF_JOB_ID + 00, 2) + '-' + 
                           SUBSTRING(@PROGRAM_NAME, @START_OF_JOB_ID + 10, 2) + 
                           SUBSTRING(@PROGRAM_NAME, @START_OF_JOB_ID + 08, 2) + '-' + 
                           SUBSTRING(@PROGRAM_NAME, @START_OF_JOB_ID + 14, 2) + 
                           SUBSTRING(@PROGRAM_NAME, @START_OF_JOB_ID + 12, 2) + '-' + 
                           SUBSTRING(@PROGRAM_NAME, @START_OF_JOB_ID + 16, 4) + '-' + 
                           SUBSTRING(@PROGRAM_NAME, @START_OF_JOB_ID + 20, 12) AS UNIQUEIDENTIFIER)
                 ELSE NULL
            END
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[P_BIGBROTHER]
SET NOEXEC ON;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[资源占用判断]
EXEC PS_BigBrother @MINUTE_FROM_NOW = 1, @QUICKSHOW = 1;/*调取此刻执行结果*/
SET NOEXEC OFF;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @COLLECTION_TIME NVARCHAR(1000);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @COLLECTION_TIME = CONVERT(CHAR(8), CURRENT_TIMESTAMP - 2, 112);
SET @COLLECTION_TIME = CONVERT(CHAR(8), CURRENT_TIMESTAMP - 1, 112);
SET @COLLECTION_TIME = CONVERT(CHAR(8), CURRENT_TIMESTAMP - 0, 112);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[RESULT]
WITH CTE
AS (SELECT (CAST(REPLACE(TEMPDB_ALLOCATIONS, ',', '') AS INT) + CAST(REPLACE(TEMPDB_CURRENT, ',', '') AS INT)) / 10000 AS TEMPDB_ALL,
            USERNAME,
            [DD HH:MM:SS.MSS], SESSION_ID, SQL_TEXT, SQL_COMMAND, LOGIN_NAME, WAIT_INFO, TRAN_LOG_WRITES, CPU, TEMPDB_ALLOCATIONS,
            TEMPDB_CURRENT, BLOCKING_SESSION_ID, READS, WRITES, PHYSICAL_READS, QUERY_PLAN, USED_MEMORY, STATUS, TRAN_START_TIME,
            IMPLICIT_TRAN, OPEN_TRAN_COUNT, PERCENT_COMPLETE, HOST_NAME, DATABASE_NAME, PROGRAM_NAME,
            CASE WHEN CHARINDEX('SQLAgent', PROGRAM_NAME, 1) > 0
                 THEN CAST((SELECT dbo.GetJobIdFromProgramName(PROGRAM_NAME)) AS VARCHAR(500))
                 ELSE PROGRAM_NAME
            END AS JOB_ID,
            START_TIME, LOGIN_TIME, REQUEST_ID, COLLECTION_TIME
     FROM XTB_BIGBROTHER WITH(NOLOCK)
     LEFT OUTER JOIN.DBO.USER_DEPT_DETAIL WITH(NOLOCK)
     ON XTB_BIGBROTHER.LOGIN_NAME = USER_DEPT_DETAIL.USERID
     WHERE COLLECTION_TIME >= @COLLECTION_TIME)
SELECT ISNULL(b.JOB_OWNER, c.USERNAME) AS WHODIDTHIS,
       --b.JOB_OWNER,
       b.name,
       --c.USERNAME,
       a.TEMPDB_ALL, a.USERNAME, a.[DD HH:MM:SS.MSS], a.SESSION_ID, a.SQL_TEXT, a.SQL_COMMAND, a.LOGIN_NAME,
       a.WAIT_INFO, a.TRAN_LOG_WRITES, a.CPU, a.TEMPDB_ALLOCATIONS, a.TEMPDB_CURRENT, a.BLOCKING_SESSION_ID, a.READS, a.WRITES,
       a.PHYSICAL_READS, a.QUERY_PLAN, a.USED_MEMORY, a.STATUS, a.TRAN_START_TIME, a.IMPLICIT_TRAN, a.OPEN_TRAN_COUNT,
       a.PERCENT_COMPLETE, a.HOST_NAME, a.DATABASE_NAME, a.PROGRAM_NAME, a.job_id, a.START_TIME, a.LOGIN_TIME, a.REQUEST_ID,
       a.COLLECTION_TIME
FROM CTE                          AS a
LEFT OUTER JOIN XTB_sysjobs_owner AS b
ON a.job_id = CAST(b.job_id AS VARCHAR(500))
LEFT OUTER JOIN dbo.USER_DEPT_DETAIL AS c WITH(NOLOCK)
ON (CASE WHEN CHARINDEX('_', a.LOGIN_NAME, 1) > 0 THEN LEFT(a.LOGIN_NAME, LEN(a.LOGIN_NAME) - 3) ELSE a.LOGIN_NAME END) = c.USERID
--ORDER BY TEMPDB_ALL DESC;
ORDER BY COLLECTION_TIME DESC;
```

<hr style="border:2px solid DeepSkyBlue"> </hr>

### 作业继任——MSYS_zuoyehuanren
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[备份数据]
IF OBJECT_ID('.dbo.SYSJOBS_BAK_SUCCESSOR', 'U') IS NOT NULL DROP TABLE SYSJOBS_BAK_SUCCESSOR;
SELECT * INTO SYSJOBS_BAK_SUCCESSOR FROM MSDB.DBO.SYSJOBS WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @KEYWORD NVARCHAR(2000);
DECLARE @SUCCESSOR_NAME NVARCHAR(2000);
DECLARE @SUCCESSOR_PHONE NVARCHAR(2000);
DECLARE @PRINT_NUM INT;
DECLARE @EXEC_NUM INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @KEYWORD = N'XHQ';
SET @SUCCESSOR_NAME = N'XHQ';
SET @SUCCESSOR_PHONE = N'13713460495';
SET @PRINT_NUM = 1;
SET @EXEC_NUM = 0;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[PRINT]
IF @PRINT_NUM > @EXEC_NUM
BEGIN
     SELECT JOB_ID, ORIGINATING_SERVER_ID, NAME, ENABLED, DESCRIPTION, START_STEP_ID, CATEGORY_ID, OWNER_SID, NOTIFY_LEVEL_EVENTLOG,
            NOTIFY_LEVEL_EMAIL, NOTIFY_LEVEL_NETSEND, NOTIFY_LEVEL_PAGE, NOTIFY_EMAIL_OPERATOR_ID, NOTIFY_NETSEND_OPERATOR_ID,
            NOTIFY_PAGE_OPERATOR_ID, DELETE_LEVEL, DATE_CREATED, DATE_MODIFIED, VERSION_NUMBER
     FROM MSDB.DBO.SYSJOBS WITH(NOLOCK)
     WHERE CHARINDEX(@KEYWORD, NAME, 1) > 0
     AND   ENABLED = 1;
END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[EXEC]
IF @PRINT_NUM < @EXEC_NUM
BEGIN
    UPDATE msdb.dbo.sysjobs
    SET NAME = REPLACE(NAME, @KEYWORD, @SUCCESSOR_NAME),
        DESCRIPTION = @SUCCESSOR_PHONE
    WHERE CHARINDEX(@KEYWORD, NAME, 1) > 0 AND ENABLED = 1;
END
```

<hr style="border:3px solid ForestGreen"> </hr>

## INFO
### 员工信息——INFO_yuangongone
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[赋值变量]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @READ INT;
DECLARE @INPUT VARCHAR(100);
DECLARE @USERNAME VARCHAR(100);
DECLARE @USERID VARCHAR(100);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
                                                                  /*#TEMPORARY#*/SET @READ = 0;
                                                                  /*#TEMPORARY#*/SET @INPUT = '邓珊|102470';
SET @USERNAME = SUBSTRING(@INPUT, 1, CHARINDEX('|', @INPUT, 1) - 1);
SET @USERID = SUBSTRING(@INPUT, CHARINDEX('|', @INPUT, 1) + 1, LEN(@INPUT) - CHARINDEX('|', @INPUT, 1));
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[获取当前使用库名]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @DBNAME VARCHAR(200);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SELECT @DBNAME = [NAME]
FROM MASTER.dbo.SYSDATABASES WITH(NOLOCK)
WHERE EXISTS (SELECT *
              FROM MASTER.dbo.SYSPROCESSES WITH(NOLOCK)
              WHERE SPID = @@SPID
              AND SYSPROCESSES.dbid = SYSDATABASES.dbid);
IF @READ = 1 BEGIN SELECT @DBNAME AS '@DBNAME' END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[某人]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[READ]
IF @READ = 1 BEGIN SELECT @USERNAME AS '@USERNAME', @USERID AS '@USERID' END;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[RESULT]
IF @DBNAME = 'odsdb' OR @DBNAME = 'odsdbfq' OR @DBNAME = 'odsdb_basic'
BEGIN
     SELECT USERID, USERNAME, POSITION, DEPT7, DEPT6, DEPT5, DEPT4, DEPT3, DEPT2, DEPT1, PHONE, RUZHITIME, JIANZHISTAT, STAT
     FROM dbo.USER_DEPT_DETAIL WITH(NOLOCK)
     WHERE(CASE WHEN LEN(@USERNAME) > 0 THEN @USERNAME ELSE USERNAME END = USERNAME)/*1=1的艺术*/
     AND  (CASE WHEN LEN(@USERNAME) = 0 THEN @USERID ELSE USERID END = USERID);/*1=1的艺术*/
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[部门]
IF LEN(@USERNAME) > 0
BEGIN
     SELECT USERID, USERNAME, POSITION, DEPT7, DEPT6, DEPT5, DEPT4, DEPT3, DEPT2, DEPT1, PHONE, RUZHITIME, JIANZHISTAT, STAT
     FROM dbo.USER_DEPT_DETAIL WITH(NOLOCK)
     WHERE USERNAME IN(SELECT USERNAME
                       FROM dbo.XTB_BI_TEAMMATE_ABBR WITH(NOLOCK)
                       WHERE (CASE WHEN LEN(@USERNAME) > 0 THEN @USERNAME ELSE USERNAME END) = BOSS)
          AND STAT = '在职'
     ORDER BY CAST(USERID AS INT) ASC;
END
```

### 工资收入——INFO_SALARY
#### 完整版—20220401
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @BADGE TABLE(NUM INT IDENTITY(1, 1), BADGE NVARCHAR(1000));
DECLARE @BADGE_INPUT NVARCHAR(1000);
DECLARE @TERM NVARCHAR(1000);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
INSERT INTO @BADGE(BADGE)
SELECT VALUE FROM STRING_SPLIT('102470', '|');
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[全表字段]
SET NOEXEC ON;
--格式1
SELECT *
FROM V_XZ WITH(NOLOCK)
WHERE BADGE IN(SELECT BADGE FROM @BADGE)
ORDER BY TERM DESC;
--格式2
SELECT TERM,/*薪资核算月*/
       BADGE,/*工号*/ NAME,/*姓名*/
       BZGZ,/*基本工资*/ CQGZ,/*出勤工资*/ WPJT,/*外派津贴*/ JXGZ,/*绩效工资*/
       QQJ,/*全勤奖*/ SLJT,/*司龄津贴*/ ZBF,/*值班费*/ QTJJ,/*其他津贴*/
       GWBT,/*高温补贴*/ JTBT,/*交通补贴*/ ZWBT,/*驻外补贴*/
       DHBT,/*电话补贴*/ HSBT,/*伙食补贴*/ TSBT,/*特殊补贴*/ GANGWBT,/*岗位补贴*/ ZXBT,/*专项补贴*/
       QTBT,/*其他补贴*/ DNBT,/*电脑补贴*/ GZTZ,/*工资调整项*/
       KBX,/*扣保险金*/
       KZFGJJ,/*扣住房公积金*/
       DKS,/*代扣税*/
       GYYJ,/*工衣押金*/ KSSZJ,/*扣宿舍租金*/ KSDF,/*扣水电费*/ BMKK,/*部门扣款*/
       KQKK,/*考勤扣款*/ DNKK,/*电脑扣款*/ GCKK,/*购车扣款*/ GFKK,/*购房扣款*/ QTJJ,/*其他借款*/
       WWCRWKK,/*未完成任务扣款*/ AXKK,/*爱心捐款*/ QTKK,/*其他扣款*/ DJSB,/*代缴社保*/
       GRJK,/*个人借款*/ TZJK,/*投资借款扣款*/
       YFGZ,/*应发工资*/ SFHJ,/*实发合计*/
       MDJBF_HD, YYYY, MM, DD, YYYY1, MM1, DD1
FROM V_XZ WITH(NOLOCK)
WHERE BADGE IN(SELECT BADGE FROM @BADGE)
ORDER BY TERM DESC
SET NOEXEC OFF;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[我用字段]
WITH CTE
AS (SELECT TERM,/*薪资核算月*/
           BADGE,/*工号*/ NAME,/*姓名*/
           BZGZ,/*基本工资*/ CQGZ,/*出勤工资*/ WPJT,/*外派津贴*/ JXGZ,/*绩效工资*/
           QQJ,/*全勤奖*/ SLJT,/*司龄津贴*/ ZBF,/*值班费*/ QTJJ,/*其他津贴*/
           --GWBT,/*高温补贴*/ JTBT,/*交通补贴*/ ZWBT,/*驻外补贴*/
           DHBT,/*电话补贴*/ HSBT,/*伙食补贴*/ TSBT,/*特殊补贴*/ GANGWBT,/*岗位补贴*/ ZXBT,/*专项补贴*/
           QTBT,/*其他补贴*/ DNBT,/*电脑补贴*/ GZTZ,/*工资调整项*/
           KBX,/*扣保险金*/
           KZFGJJ,/*扣住房公积金*/
           DKS,/*代扣税*/
           --GYYJ,/*工衣押金*/ KSSZJ,/*扣宿舍租金*/ KSDF,/*扣水电费*/ BMKK,/*部门扣款*/
           KQKK,/*考勤扣款*/
           --DNKK,/*电脑扣款*/ GCKK,/*购车扣款*/ GFKK,/*购房扣款*/ QTJJ,/*其他借款*/
           --WWCRWKK,/*未完成任务扣款*/ AXKK,/*爱心捐款*/ QTKK,/*其他扣款*/ DJSB,/*代缴社保*/
           --GRJK,/*个人借款*/ TZJK,/*投资借款扣款*/
           YFGZ,/*应发工资*/ SFHJ,/*实发合计*/
           MDJBF_HD, YYYY, MM, DD, YYYY1, MM1, DD1,
           (CQGZ + JXGZ + QQJ + SLJT + ZBF + QTJJ + DHBT + HSBT + TSBT + GANGWBT + ZXBT + DNBT + GZTZ + KQKK/* + KBX + KZFGJJ + DKS */- YFGZ) AS MYCAL
    FROM V_XZ WITH(NOLOCK)
    WHERE BADGE IN(SELECT BADGE FROM @BADGE))
SELECT *
FROM CTE
--WHERE MYCAL != 0
ORDER BY TERM DESC
```

#### 简化版—20220401
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @BADGE TABLE(NUM INT IDENTITY(1, 1), BADGE NVARCHAR(1000));
DECLARE @BADGE_INPUT NVARCHAR(1000);
DECLARE @TERM NVARCHAR(1000);
DECLARE @PERFORMANCE_BASIC INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
INSERT INTO @BADGE(BADGE)
SELECT VALUE FROM STRING_SPLIT('102470', '|');
SET @PERFORMANCE_BASIC = 1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[全表字段]
SET NOEXEC ON;
--格式1
SELECT *
FROM V_XZ WITH(NOLOCK)
WHERE BADGE IN(SELECT BADGE FROM @BADGE)
ORDER BY TERM DESC;
--格式2
SELECT TERM,/*薪资核算月*/
       BADGE,/*工号*/ NAME,/*姓名*/
       BZGZ,/*基本工资*/ CQGZ,/*出勤工资*/ WPJT,/*外派津贴*/ JXGZ,/*绩效工资*/
       QQJ,/*全勤奖*/ SLJT,/*司龄津贴*/ ZBF,/*值班费*/ QTJJ,/*其他津贴*/
       GWBT,/*高温补贴*/ JTBT,/*交通补贴*/ ZWBT,/*驻外补贴*/
       DHBT,/*电话补贴*/ HSBT,/*伙食补贴*/ TSBT,/*特殊补贴*/ GANGWBT,/*岗位补贴*/ ZXBT,/*专项补贴*/
       QTBT,/*其他补贴*/ DNBT,/*电脑补贴*/ GZTZ,/*工资调整项*/
       KBX,/*扣保险金*/
       KZFGJJ,/*扣住房公积金*/
       DKS,/*代扣税*/
       GYYJ,/*工衣押金*/ KSSZJ,/*扣宿舍租金*/ KSDF,/*扣水电费*/ BMKK,/*部门扣款*/
       KQKK,/*考勤扣款*/ DNKK,/*电脑扣款*/ GCKK,/*购车扣款*/ GFKK,/*购房扣款*/ QTJJ,/*其他借款*/
       WWCRWKK,/*未完成任务扣款*/ AXKK,/*爱心捐款*/ QTKK,/*其他扣款*/ DJSB,/*代缴社保*/
       GRJK,/*个人借款*/ TZJK,/*投资借款扣款*/
       YFGZ,/*应发工资*/ SFHJ,/*实发合计*/
       MDJBF_HD, YYYY, MM, DD, YYYY1, MM1, DD1
FROM V_XZ WITH(NOLOCK)
WHERE BADGE IN(SELECT BADGE FROM @BADGE)
ORDER BY TERM DESC
SET NOEXEC OFF;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[我用字段]
WITH CTE
AS (SELECT CONVERT(CHAR(6), TERM, 112) AS 月份,
           BADGE AS 工号,
           NAME AS 姓名,
           CQGZ AS 出勤工资,
           (CASE WHEN TERM >= '20180101' AND TERM < '20181031' THEN '1200'
                 WHEN TERM >= '20181031' AND TERM < '20190101' THEN '1800'
                 WHEN TERM >= '20190101' AND TERM < '20190901' THEN '2400'
                 WHEN TERM >= '20190901' AND TERM < '20220101' THEN '2500'
                 WHEN TERM >= '20220101' AND TERM < '20300101' THEN '2600'
           END) AS 绩效基础,
           (JXGZ * 1.00) / (CASE WHEN TERM >= '20180101' AND TERM < '20181031' THEN '1200'
                                 WHEN TERM >= '20181031' AND TERM < '20190101' THEN '1800'
                                 WHEN TERM >= '20190101' AND TERM < '20190901' THEN '2400'
                                 WHEN TERM >= '20190901' AND TERM < '20220101' THEN '2500'
                                 WHEN TERM >= '20220101' AND TERM < '20300101' THEN '2600'
                            END) AS 绩效点数,
           --(JXGZ * 1.00) / @PERFORMANCE_BASIC AS 绩效点数,
           JXGZ AS 绩效工资,
           QQJ AS 全勤,
           (SLJT + ZBF + QTJJ + DHBT + HSBT + TSBT + GANGWBT + ZXBT + DNBT + GZTZ) AS 补贴,
           (KQKK + KBX + KZFGJJ + DKS) AS 扣款,
           KQKK AS 考勤扣款,
           YFGZ AS 应发工资,
           SFHJ AS 实发合计
    FROM V_XZ WITH(NOLOCK)
    WHERE BADGE IN(SELECT BADGE FROM @BADGE))
SELECT 月份, 工号, 姓名, 出勤工资, 绩效基础, 绩效点数, 绩效工资, 全勤, 补贴, 扣款, 应发工资, 实发合计,
       (出勤工资 + 绩效工资 + 全勤 + 补贴 + 考勤扣款 - 应发工资) AS MYCAL
FROM CTE
WHERE 月份 > '20211231'
AND   月份 < '20221231'
ORDER BY 月份 DESC;
```

#### 简化版—20220608
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[PASSWORD]
--SERVER:192.168.5.109
--LOGINNAME:HR_DATA
--PASSWORD:hrdata123*
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @BADGE TABLE(NUM INT IDENTITY(1, 1), BADGE NVARCHAR(1000));
DECLARE @BADGE_INPUT NVARCHAR(1000);
DECLARE @TERM NVARCHAR(1000);
DECLARE @PERFORMANCE_BASIC INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
INSERT INTO @BADGE(BADGE)
SELECT VALUE FROM STRING_SPLIT('102470', '|');
SET @PERFORMANCE_BASIC = 1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[全表字段]
SET NOEXEC ON;
--格式1
SELECT *
FROM V_XZ WITH(NOLOCK)
WHERE BADGE IN(SELECT BADGE FROM @BADGE)
ORDER BY TERM DESC;
--格式2
SELECT TERM,/*薪资核算月*/
       BADGE,/*工号*/ NAME,/*姓名*/
       BZGZ,/*基本工资*/ CQGZ,/*出勤工资*/ WPJT,/*外派津贴*/ JXGZ,/*绩效工资*/
       QQJ,/*全勤奖*/ SLJT,/*司龄津贴*/ ZBF,/*值班费*/ QTJJ,/*其他津贴*/
       GWBT,/*高温补贴*/ JTBT,/*交通补贴*/ ZWBT,/*驻外补贴*/
       DHBT,/*电话补贴*/ HSBT,/*伙食补贴*/ TSBT,/*特殊补贴*/ GANGWBT,/*岗位补贴*/ ZXBT,/*专项补贴*/
       QTBT,/*其他补贴*/ DNBT,/*电脑补贴*/ GZTZ,/*工资调整项*/
       KBX,/*扣保险金*/
       KZFGJJ,/*扣住房公积金*/
       DKS,/*代扣税*/
       GYYJ,/*工衣押金*/ KSSZJ,/*扣宿舍租金*/ KSDF,/*扣水电费*/ BMKK,/*部门扣款*/
       KQKK,/*考勤扣款*/ DNKK,/*电脑扣款*/ GCKK,/*购车扣款*/ GFKK,/*购房扣款*/ QTJJ,/*其他借款*/
       WWCRWKK,/*未完成任务扣款*/ AXKK,/*爱心捐款*/ QTKK,/*其他扣款*/ DJSB,/*代缴社保*/
       GRJK,/*个人借款*/ TZJK,/*投资借款扣款*/
       YFGZ,/*应发工资*/ SFHJ,/*实发合计*/
       MDJBF_HD, YYYY, MM, DD, YYYY1, MM1, DD1
FROM V_XZ WITH(NOLOCK)
WHERE BADGE IN(SELECT BADGE FROM @BADGE)
ORDER BY TERM DESC
SET NOEXEC OFF;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[我用字段]
WITH CTE
AS (SELECT CONVERT(CHAR(6), TERM, 112) AS 月份,
           BADGE AS 工号,
           NAME AS 姓名,
           CQGZ AS 出勤工资,
           (CASE WHEN TERM >= '20180101' AND TERM < '20181031' THEN '1200'
                 WHEN TERM >= '20181031' AND TERM < '20190101' THEN '1800'
                 WHEN TERM >= '20190101' AND TERM < '20190901' THEN '2400'
                 WHEN TERM >= '20190901' AND TERM < '20220101' THEN '2500'
                 WHEN TERM >= '20220101' AND TERM < '20220501' THEN '2600'
                 WHEN TERM >= '20220501' AND TERM < '20300101' THEN '4000'
           END) AS 绩效基础,
           (JXGZ * 1.00) / (CASE WHEN TERM >= '20180101' AND TERM < '20181031' THEN '1200'
                                 WHEN TERM >= '20181031' AND TERM < '20190101' THEN '1800'
                                 WHEN TERM >= '20190101' AND TERM < '20190901' THEN '2400'
                                 WHEN TERM >= '20190901' AND TERM < '20220101' THEN '2500'
                                 WHEN TERM >= '20220101' AND TERM < '20220501' THEN '2600'
                                 WHEN TERM >= '20220501' AND TERM < '20300101' THEN '4000'
                            END) AS 绩效点数,
           --(JXGZ * 1.00) / @PERFORMANCE_BASIC AS 绩效点数,
           JXGZ AS 绩效工资,
           QQJ AS 全勤,
           (SLJT + ZBF + QTJJ + DHBT + HSBT + TSBT + GANGWBT + ZXBT + DNBT + GZTZ) AS 补贴,
           (KQKK + KBX + KZFGJJ + DKS) AS 扣款,
           KQKK AS 考勤扣款,
           YFGZ AS 应发工资,
           SFHJ AS 实发合计
    FROM V_XZ WITH(NOLOCK)
    WHERE BADGE IN(SELECT BADGE FROM @BADGE))
SELECT 月份, 工号, 姓名, 出勤工资, 绩效基础, 绩效点数, 绩效工资, 全勤, 补贴, 扣款, 应发工资, 实发合计,
       (出勤工资 + 绩效工资 + 全勤 + 补贴 + 考勤扣款 - 应发工资) AS MYCAL
FROM CTE
WHERE 月份 > '20211231'
AND   月份 < '20221231'
ORDER BY 月份 DESC;
```
<hr style="border:3px solid ForestGreen"> </hr>

## PP
### 中枢工具箱参数调用模板——PP_USE_TOOL
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[设置变量]
DECLARE @BREAKPOINT_INPUT_II INT;
DECLARE @Build_On_People_INPUT_SD NVARCHAR(MAX);
DECLARE @MONITOR_SEND_TO_INPUT_SD NVARCHAR(MAX);
DECLARE @BREAKPOINT_INPUT_SD INT;
DECLARE @READ INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[调试版本]
SET @BREAKPOINT_INPUT_II = 1;
SET @READ = 1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[正式执行]
--SET @BREAKPOINT_INPUT_II = @BREAKPOINT_INPUT_II_ST;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[@Build_On_People_INPUT_SD]
WITH ADDTIME_MAX
AS (SELECT PROCNAME, PARMNAME, MAX(ADDTIME) AS ADDTIME_MAX
    FROM dbo.XTB_PARM_WAREHOUSE WITH(NOLOCK)
    WHERE PROCNAME = 'P_STAFF_NAMEFORSHORT'
      AND PARMNAME = '@Build_On_People_INPUT'
    GROUP BY PROCNAME, PARMNAME)
SELECT @Build_On_People_INPUT_SD = PARMVALUE
FROM dbo.XTB_PARM_WAREHOUSE AS a WITH(NOLOCK)
INNER JOIN ADDTIME_MAX AS b WITH(NOLOCK)
ON a.PROCNAME = b.PROCNAME
AND a.PARMNAME = b.PARMNAME
AND a.ADDTIME = b.ADDTIME_MAX;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[@MONITOR_SEND_TO_INPUT_SD]
WITH ADDTIME_MAX
AS (SELECT PROCNAME, PARMNAME, MAX(ADDTIME) AS ADDTIME_MAX
    FROM dbo.XTB_PARM_WAREHOUSE WITH(NOLOCK)
    WHERE PROCNAME = 'P_STAFF_NAMEFORSHORT'
      AND PARMNAME = '@MONITOR_SEND_TO_INPUT'
    GROUP BY PROCNAME, PARMNAME)
SELECT @MONITOR_SEND_TO_INPUT_SD = PARMVALUE
FROM dbo.XTB_PARM_WAREHOUSE AS a WITH(NOLOCK)
INNER JOIN ADDTIME_MAX AS b WITH(NOLOCK)
ON a.PROCNAME = b.PROCNAME
AND a.PARMNAME = b.PARMNAME
AND a.ADDTIME = b.ADDTIME_MAX;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[@BREAKPOINT_INPUT_SD]
WITH ADDTIME_MAX
AS (SELECT PROCNAME, PARMNAME, MAX(ADDTIME) AS ADDTIME_MAX
    FROM dbo.XTB_PARM_WAREHOUSE WITH(NOLOCK)
    WHERE PROCNAME = 'P_STAFF_NAMEFORSHORT'
      AND PARMNAME = '@BREAKPOINT_INPUT'
    GROUP BY PROCNAME, PARMNAME)
SELECT @BREAKPOINT_INPUT_SD = CAST(PARMVALUE AS INT)
FROM dbo.XTB_PARM_WAREHOUSE AS a WITH(NOLOCK)
INNER JOIN ADDTIME_MAX AS b WITH(NOLOCK)
ON a.PROCNAME = b.PROCNAME
AND a.PARMNAME = b.PARMNAME
AND a.ADDTIME = b.ADDTIME_MAX;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查看参数]
IF @READ = 1
BEGIN
     SELECT @Build_On_People_INPUT_SD AS '@Build_On_People_INPUT_SD',
            @MONITOR_SEND_TO_INPUT_SD AS '@MONITOR_SEND_TO_INPUT_SD',
            @BREAKPOINT_INPUT_SD      AS '@BREAKPOINT_INPUT_SD';
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[部门人员姓名获取]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[调试版本]
--EXEC p_staff_nameforshort @Build_On_People_INPUT = N'邓珊|徐海权|赵必胜|陈浩平|周焕焕',
--                          @MONITOR_SEND_TO_INPUT = N'102470',
--                          @BREAKPOINT_INPUT = 0;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[正式执行]
EXEC p_staff_nameforshort @Build_On_People_INPUT = @Build_On_People_INPUT_SD,
                          @MONITOR_SEND_TO_INPUT = @MONITOR_SEND_TO_INPUT_SD,
                          @BREAKPOINT_INPUT = @BREAKPOINT_INPUT_SD
```