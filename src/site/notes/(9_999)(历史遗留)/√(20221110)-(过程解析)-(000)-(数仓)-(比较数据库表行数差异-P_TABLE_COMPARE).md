---
{"dg-publish":true,"permalink":"/(9_999)(历史遗留)/√(20221110)-(过程解析)-(000)-(数仓)-(比较数据库表行数差异-P_TABLE_COMPARE)/"}
---


# <font color=#DC143C>(20221110)-(过程解析)-(000)-(数仓)-(比较数据库表行数差异-P_TABLE_COMPARE)</font>
URL:: 

```
dataview
table without id 入榜亮点, 入榜输出
where contains(TITLES, "")
```

```
dataview
table without id 萃取重点, 萃取难点, 萃取锚点, 萃取输出
where contains(TITLES, "")
```

## (P_TABLE_COMPARE)-(20221110)-(000)-(工具)
```SQL
USE [odsdb]
GO
/****** Object:StoredProcedure [dbo].[P_TABLE_COMPARE]    Script Date:2019-04-22 15:44:53 ******//*[2]❆❆❆❆❆*/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROC [dbo].[P_TABLE_COMPARE]
(@DATABASE_A_INPUT NVARCHAR(1000),--数据库A(本地)
 @DATABASE_B_INPUT NVARCHAR(1000),--数据库B(跨服务器)
 @READ_INPUT INT = 1,--1:全部数据|表名排序--2:两边共存|表名排序--3:特定搜索|表名排序--4:打印结果表名)
 @KEYWORD_INPUT NVARCHAR(1000) = '')
AS
BEGIN
SET NOCOUNT ON
--*******************************************************************************************************************************************************************
--提出人员:徐海权
--创建人员:徐海权
--创建时间:20221109
--创建用途:比较数据库表行数差异
--更新频率:调用执行
--修改人员:
--报表维度:
--相关过程:
--生成结果:SELECT * FROM dbo.XTB_SPACECOMPARE WITH(NOLOCK)
--*******************************************************************************************************************************************************************
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @DATABASE_A NVARCHAR(1000);
DECLARE @DATABASE_B NVARCHAR(1000);
DECLARE @TABLENAME_SERVER_A NVARCHAR(100);
DECLARE @TABLENAME_SERVER_B NVARCHAR(100);
DECLARE @TABLENAME_DATABASE_A NVARCHAR(100);
DECLARE @TABLENAME_DATABASE_B NVARCHAR(100);
DECLARE @CMD_SPACEUSED_A NVARCHAR(MAX);
DECLARE @CMD_SPACEUSED_B NVARCHAR(MAX);
DECLARE @SHOW_RESULT NVARCHAR(MAX);
DECLARE @READ INT;
DECLARE @KEYWORD NVARCHAR(2000);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @DATABASE_A = N'DATABASE_61.odsdbfq2022';
SET @DATABASE_B = N'DATABASE_6_5.odsdbfq2022';
SET @READ = 1;--1:全部数据|表名排序--2:两边共存|表名排序--3:特定搜索|表名排序--4:打印结果表名
SET @KEYWORD = N'MOUTDRPT';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET_INPUT]
SET @DATABASE_A = @DATABASE_A_INPUT
SET @DATABASE_B = @DATABASE_B_INPUT
SET @READ = @READ_INPUT
SET @KEYWORD = @KEYWORD_INPUT
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET_GET]
SET @TABLENAME_SERVER_A = SUBSTRING(@DATABASE_A, 1, CHARINDEX('.', @DATABASE_A, 1) - 1);
SET @TABLENAME_SERVER_B = SUBSTRING(@DATABASE_B, 1, CHARINDEX('.', @DATABASE_B, 1) - 1);
SET @TABLENAME_DATABASE_A = SUBSTRING(@DATABASE_A,
                                      CHARINDEX('.', @DATABASE_A, 1) + 1,
                                      LEN(@DATABASE_A) - CHARINDEX('.', @DATABASE_A, 1));
SET @TABLENAME_DATABASE_B = SUBSTRING(@DATABASE_B,
                                      CHARINDEX('.', @DATABASE_B, 1) + 1,
                                      LEN(@DATABASE_B) - CHARINDEX('.', @DATABASE_B, 1));
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[调用工具过程-P_TB_SPACEUSED]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[LOCAL]
SET @CMD_SPACEUSED_A = N'EXEC ' + @TABLENAME_DATABASE_A + N'.dbo.P_TB_SPACEUSED' + CHAR(13) + CHAR(10) + 
                       N'@TABLENAME_PICKUP_INPUT = NULL,' + CHAR(13) + CHAR(10) + 
                       N'@SHOWORNOT_INPUT = 0';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[EXEC]
EXEC SP_EXECUTESQL @STMT = @CMD_SPACEUSED_A;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CROSS SERVER]
SET @CMD_SPACEUSED_B = N'EXEC ' + @DATABASE_B + N'.dbo.P_TB_SPACEUSED' + CHAR(13) + CHAR(10) + 
                       N'@TABLENAME_PICKUP_INPUT = NULL,' + CHAR(13) + CHAR(10) + 
                       N'@SHOWORNOT_INPUT = 0';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[EXEC]
EXEC SP_EXECUTESQL @STMT = @CMD_SPACEUSED_B;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[汇总数据]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[RULE]
SET @SHOW_RESULT = N'IF OBJECT_ID(''.dbo.XTB_SPACECOMPARE'', ''U'') IS NOT NULL DROP TABLE dbo.XTB_SPACECOMPARE;' + CHAR(13) + CHAR(10) + 
                   N'WITH TableName_ALL' + CHAR(13) + CHAR(10) + 
                   N'AS (SELECT TableName FROM ' + @TABLENAME_DATABASE_A + '.dbo.XTB_SPACEUSED WITH(NOLOCK)' + CHAR(13) + CHAR(10) + 
                   SPACE(4) + N'UNION' + CHAR(13) + CHAR(10) + 
                   SPACE(4) + N'SELECT TableName FROM ' + @DATABASE_B + '.dbo.XTB_SPACEUSED WITH(NOLOCK))' + CHAR(13) + CHAR(10) + 
                   N'SELECT a.TableName,' + CHAR(13) + CHAR(10) + 
                   N'''' + @TABLENAME_DATABASE_A + '''' + SPACE(1) + 'AS TABLENAME_a,' + CHAR(13) + CHAR(10) + 
                   N'b.CreateDate AS a_CreateDate, b.RowCounts AS a_RowCounts, b.TotalSpaceMB AS a_TotalSpaceMB,' + CHAR(13) + CHAR(10) + 
                   N'''' + @DATABASE_B + '''' + SPACE(1) + 'AS TABLENAME_b,' + CHAR(13) + CHAR(10) + 
                   N'c.CreateDate AS b_CreateDate, c.RowCounts AS b_RowCounts, c.TotalSpaceMB AS b_TotalSpaceMB,' + CHAR(13) + CHAR(10) + 
                   N'(ISNULL(b.RowCounts, 0) - ISNULL(c.RowCounts, 0)) AS tableA_minus_tableB,' + CHAR(13) + CHAR(10) + 
                   N'CASE WHEN (b.RowCounts + c.RowCounts) IS NULL THEN ''单边存在'' ELSE NULL END AS ONE_SIDE_MARK' + CHAR(13) + CHAR(10) + 
                   N'INTO dbo.XTB_SPACECOMPARE' + CHAR(13) + CHAR(10) + 
                   N'FROM TableName_ALL AS a WITH(NOLOCK)' + CHAR(13) + CHAR(10) + 
                   N'LEFT OUTER JOIN ' + @TABLENAME_DATABASE_A + '.dbo.XTB_SPACEUSED AS b WITH(NOLOCK)' + CHAR(13) + CHAR(10) + 
                   N'ON a.TableName = b.TableName' + CHAR(13) + CHAR(10) + 
                   N'LEFT OUTER JOIN ' + @DATABASE_B + '.dbo.XTB_SPACEUSED AS c WITH(NOLOCK)' + CHAR(13) + CHAR(10) + 
                   N'ON a.TableName = c.TableName;'
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[EXEC]
EXEC SP_EXECUTESQL @STMT = @SHOW_RESULT;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[展示数据]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[全部数据|表名排序]
IF @READ = 1
BEGIN
     SELECT TableName,
            TABLENAME_a, a_CreateDate, a_RowCounts, a_TotalSpaceMB,
            TABLENAME_b, b_CreateDate, b_RowCounts, b_TotalSpaceMB,
            tableA_minus_tableB, ONE_SIDE_MARK
     FROM dbo.XTB_SPACECOMPARE WITH(NOLOCK)
     ORDER BY TableName
END;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[两边共存|表名排序]
IF @READ = 2
BEGIN
     SELECT TableName,
            TABLENAME_a, a_CreateDate, a_RowCounts, a_TotalSpaceMB,
            TABLENAME_b, b_CreateDate, b_RowCounts, b_TotalSpaceMB,
            tableA_minus_tableB, ONE_SIDE_MARK
     FROM dbo.XTB_SPACECOMPARE WITH(NOLOCK)
     WHERE ONE_SIDE_MARK IS NULL
     ORDER BY TableName
END;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[特定搜索|表名排序]
IF @READ = 3
BEGIN
     SELECT TableName,
            TABLENAME_a, a_CreateDate, a_RowCounts, a_TotalSpaceMB,
            TABLENAME_b, b_CreateDate, b_RowCounts, b_TotalSpaceMB,
            tableA_minus_tableB, ONE_SIDE_MARK
     FROM dbo.XTB_SPACECOMPARE WITH(NOLOCK)
     WHERE CHARINDEX(@KEYWORD, TableName, 1) > 0
     ORDER BY TableName
END;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[打印结果表名]
IF @READ = 4
BEGIN
     SELECT '结果表名:SELECT * FROM dbo.XTB_SPACECOMPARE WITH(NOLOCK)'
END
--*******************************************************************************************************************************************************************
SET NOCOUNT OFF
END
```