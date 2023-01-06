---
{"dg-publish":true,"permalink":"/(2_021)(算法解析)/√(20221024)-(过程解析)-(000)-(数仓)-(替身更新-P_TABLE_UPDATE)/"}
---


# <font color=#DC143C>(20221024)-(过程解析)-(000)-(数仓)-(替身更新P_TABLE_UPDATE)</font>
URL:: 

```
dataview
table without id 入榜亮点, 入榜输出
where contains(TITLES, "")
```

| 萃取重点 | 萃取难点 | 萃取锚点 | 萃取输出 |
| ---- | ---- | ---- | ---- |
| \-   | \-   | \-   | \-   |


## 测试案例
```SQL
EXEC P_TABLE_UPDATE
@TABLE_NEEDTO_UPDATE = N'odsdb2022.dbo.moutdrpt_20221022',/*待更新表名*/
@TABLENAME_SYNC_INPUT = N'newbuy.dbo.TMP_MOUTDRPT_C_GOODS',/*新数据表名*/
@CONDITION_JOIN_INPUT = N'a.ADATE = b.ADATE AND a.ASTORE = b.ASTORE AND a.BGDGID = b.BGDGID',/*待更新+新数据→找保留部分关联条件*/
@CONDITION_FILTER_INPUT = N'ADATE = CONVERT(CHAR(8), CURRENT_TIMESTAMP - 2, 112)',/*新数据非全量更新过滤条件*/
@SERVER_NAME_FOR_DROP = N'DATABASE_61',/*预约删除需要服务器名字*/
@BREAKPOINT_INPUT = 1/*|10执行并调试|1执行|*/
```

## 逻辑压缩
+ 参数说明
    + 1-@TABLE_NEEDTO_UPDATE：待更新表名`DATABASENAME.DBO.TABLENAME`
    + 2-@TABLENAME_SYNC_INPUT：新数据表名`DATABASENAME.DBO.TABLENAME`
    + 3-@CONDITION_JOIN_INPUT：参数1(待更新)和参数2(新数据)→找保留部分关联条件
    + 4-@CONDITION_FILTER_INPUT：新数据非全量更新过滤条件
+ 找到数据保留部分(也就是不更新数据)
    + 拼接形式调用工具过程`P_XHEADGET`，生成原表列名(<strong><font color=#E6E022>带别名前缀</font></strong>)
        + 存入表变量
    + 生成`保留部分新表①`，格式`@TABLENAME_DATABASE_ORIGIN + N'.dbo.' + @TABLENAME_TABLE_ORIGIN + N'_SAVE_' + CONVERT(CHAR(8), CURRENT_TIMESTAMP, 112)`
+ 找到数据更新部分
    + 拼接形式<strong><font color=#9966CC>调用工具过程</font></strong>`P_XHEADGET`，生成原表列名(<strong><font color=#E6E022>不带别名前缀</font></strong>)
        + 存入表变量
    + 插入`保留部分新表①`
        + 数据来源：@TABLENAME_SYNC
        + 取值范围：`CASE WHEN LEN(@CONDITION_FILTER) > 0 THEN 'WHERE' + SPACE(1) + @CONDITION_FILTER ELSE '' END`
+ 复制索引
    + <strong><font color=#9966CC>调用工具过程</font></strong>`P_INDEX_CLONE`
+ 交换表名
    + 系统表`SYSOBJECTS`判断替换新表是否存在
    + `SELECT TOP 1 * FROM`判断新表数据量是否异常
    + 执行换名：`@TABLENAME_TABLE_ORIGIN`→`@TABLENAME_TABLE_ORIGIN + '_STAND_IN'''`
    + 执行换名：`@TABLENAME_TABLE_ORIGIN + N'_SAVE_' + CONVERT(CHAR(8), CURRENT_TIMESTAMP, 112)`→`@TABLENAME_TABLE_ORIGIN`
+ 预约删除
    + 插入登记表`odsdb.dbo.XTB_SCHEDULED_TABLE_DROP`，删除时间为明天`CONVERT(CHAR(8), DATEADD(DAY, 1, CURRENT_TIMESTAMP), 112)`
    + <strong><font color=#9966CC>调用工具过程</font></strong>`P_TABLE_DROP`

## (P_TABLE_UPDATE)-(20221024)-(000)-(工具)
```SQL
USE [odsdb]
GO
/****** Object:  StoredProcedure [dbo].[P_TABLE_UPDATE]    Script Date: 2022/10/24 11:41:39 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROC [dbo].[P_TABLE_UPDATE]
(@TABLE_NEEDTO_UPDATE NVARCHAR(200),/*待更新表名*/
 @TABLENAME_SYNC_INPUT NVARCHAR(200),/*新数据表名*/
 @CONDITION_JOIN_INPUT NVARCHAR(1000),/*待更新+新数据→找保留部分关联条件*/
 @CONDITION_FILTER_INPUT NVARCHAR(1000),/*新数据非全量更新过滤条件*/
 @SERVER_NAME_FOR_DROP NVARCHAR(1000) = N'DATABASE_61',/*预约删除需要服务器名字*/
 @BREAKPOINT_INPUT INT = 1)/*|10执行并调试|1执行|*/
AS
BEGIN
SET NOCOUNT ON
--*******************************************************************************************************************************************************************
--提出人员:徐海权
--创建人员:徐海权
--创建时间:20221020
--创建用途:大表新表写入方式更新
--更新频率:调用方式
--修改人员:
--报表维度:
--生成结果:
--修改时间:20221024(徐海权)
--*******************************************************************************************************************************************************************
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[定义变量]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @BREAKPOINT INT;
DECLARE @TABLENAME_ORIGIN NVARCHAR(200);
DECLARE @TABLENAME_SYNC NVARCHAR(200);
DECLARE @TABLENAME_SAVE NVARCHAR(200);
DECLARE @TABLENAME_DATABASE_ORIGIN NVARCHAR(100);
DECLARE @TABLENAME_DATABASE_SYNC NVARCHAR(100);
DECLARE @TABLENAME_TABLE_ORIGIN NVARCHAR(100);
DECLARE @TABLENAME_TABLE_SYNC NVARCHAR(100);
DECLARE @CMD_1 NVARCHAR(MAX);
DECLARE @CMD_2 NVARCHAR(MAX);
DECLARE @CMD_3 NVARCHAR(MAX);
DECLARE @CMD_4 NVARCHAR(MAX);
DECLARE @CMD_5 NVARCHAR(MAX);
DECLARE @COLUMNNAME_ORIGIN TABLE(COLUMNNAME_ORIGIN NVARCHAR(MAX));
DECLARE @CONDITION_JOIN NVARCHAR(1000);
DECLARE @CONDITION_FILTER NVARCHAR(1000);
DECLARE @SERVER_NAME NVARCHAR(1000);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET_TEST]
SET NOEXEC ON;
SET @BREAKPOINT = 10;/*|10执行并调试|1执行|*/
SET @TABLENAME_ORIGIN = N'odsdb2022.dbo.moutdrpt_20221020';--待更新表
SET @TABLENAME_SYNC = N'newbuy.dbo.TMP_MOUTDRPT_C_GOODS';--更新源头
SET @CONDITION_JOIN = N'a.ADATE = b.ADATE AND a.ASTORE = b.ASTORE AND a.BGDGID = b.BGDGID';--更新源头CONNECT待更新表|关联条件
SET @CONDITION_FILTER = N'ADATE = CONVERT(CHAR(8), CURRENT_TIMESTAMP - 2, 112)'
SET @SERVER_NAME = N'DATABASE_61';
SET NOEXEC OFF;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET_INPUT]
SET @BREAKPOINT = @BREAKPOINT_INPUT;
SET @TABLENAME_ORIGIN = @TABLE_NEEDTO_UPDATE;
SET @TABLENAME_SYNC = @TABLENAME_SYNC_INPUT;
SET @CONDITION_JOIN = @CONDITION_JOIN_INPUT;
SET @CONDITION_FILTER = @CONDITION_FILTER_INPUT;
SET @SERVER_NAME = @SERVER_NAME_FOR_DROP;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET_GET]
SET @TABLENAME_DATABASE_ORIGIN = SUBSTRING(@TABLENAME_ORIGIN, 1, CHARINDEX('.', @TABLENAME_ORIGIN, 1) - 1);--库名
SET @TABLENAME_DATABASE_SYNC = SUBSTRING(@TABLENAME_SYNC, 1, CHARINDEX('.', @TABLENAME_SYNC, 1) - 1);--库名
SET @TABLENAME_TABLE_ORIGIN = REVERSE(SUBSTRING(REVERSE(@TABLENAME_ORIGIN), 1, CHARINDEX('.', REVERSE(@TABLENAME_ORIGIN), 1) - 1));--表名
SET @TABLENAME_TABLE_SYNC = REVERSE(SUBSTRING(REVERSE(@TABLENAME_SYNC), 1, CHARINDEX('.', REVERSE(@TABLENAME_SYNC), 1) - 1));--表名
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[READ]
IF @BREAKPOINT >= 10
BEGIN
     SELECT @TABLENAME_DATABASE_ORIGIN AS '@TABLENAME_DATABASE_ORIGIN',
            @TABLENAME_DATABASE_SYNC AS '@TABLENAME_DATABASE_SYNC',
            @TABLENAME_TABLE_ORIGIN AS '@TABLENAME_TABLE_ORIGIN',
            @TABLENAME_TABLE_SYNC AS '@TABLENAME_TABLE_SYNC'
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[数据|保留部分]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[动态拼接|原表列名(带别名前缀)]
SET @CMD_1 = N'EXEC' + SPACE(1) + @TABLENAME_DATABASE_ORIGIN + N'.dbo.P_XHEADGET' + SPACE(1) + 
             N'''' + @TABLENAME_TABLE_ORIGIN + N'''' + N', ''a''';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[执行拼接|@CMD_1]
INSERT INTO @COLUMNNAME_ORIGIN EXEC SP_EXECUTESQL @STMT = @CMD_1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[动态拼接|生成数据]
SET @CMD_2 = N'IF OBJECT_ID(''' + @TABLENAME_DATABASE_ORIGIN + N'.dbo.' + @TABLENAME_TABLE_ORIGIN + N'_SAVE_' + CONVERT(CHAR(8), CURRENT_TIMESTAMP, 112) + 
             N''', ''U'') IS NOT NULL DROP TABLE ' + @TABLENAME_DATABASE_ORIGIN + 
             N'.dbo.' + @TABLENAME_TABLE_ORIGIN + N'_SAVE_' + CONVERT(CHAR(8), CURRENT_TIMESTAMP, 112) + CHAR(13) + CHAR(10) + 
             N'SELECT' + SPACE(1) + (SELECT COLUMNNAME_ORIGIN FROM @COLUMNNAME_ORIGIN) + CHAR(13) + CHAR(10) + 
             N'INTO' + SPACE(1) + @TABLENAME_DATABASE_ORIGIN + N'.dbo.' + @TABLENAME_TABLE_ORIGIN + 
             N'_SAVE_' + CONVERT(CHAR(8), CURRENT_TIMESTAMP, 112) + CHAR(13) + CHAR(10) + 
             N'FROM ' + @TABLENAME_ORIGIN + N' AS a WITH(NOLOCK)' + CHAR(13) + CHAR(10) + 
             N'WHERE NOT EXISTS(SELECT * FROM ' + @TABLENAME_SYNC + N' AS b WITH(NOLOCK)' + CHAR(13) + CHAR(10) + SPACE(17) + 
             N'WHERE ' + @CONDITION_JOIN + N');';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[执行拼接|@CMD_2]
IF @BREAKPOINT >= 10--打印
BEGIN
     PRINT @CMD_2;
END
IF @BREAKPOINT >= 1--执行
BEGIN
     EXEC SP_EXECUTESQL @STMT = @CMD_2;
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[数据|更新部分]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[动态拼接|原表列名(不带别名前缀)]
SET @CMD_3 = N'EXEC' + SPACE(1) + @TABLENAME_DATABASE_ORIGIN + N'.dbo.P_XHEADGET' + SPACE(1) + 
             N'''' + @TABLENAME_TABLE_ORIGIN + N'''' + N', ''''';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[执行拼接|@CMD_3]
--∷∷∷∷∷∷[清除历史]
DELETE FROM @COLUMNNAME_ORIGIN;
--∷∷∷∷∷∷[写入新列名]
INSERT INTO @COLUMNNAME_ORIGIN EXEC SP_EXECUTESQL @STMT = @CMD_3;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[动态拼接|生成数据]
SET @CMD_4 = N'INSERT INTO ' + @TABLENAME_DATABASE_ORIGIN + N'.dbo.' + @TABLENAME_TABLE_ORIGIN + N'_SAVE_' + CONVERT(CHAR(8), CURRENT_TIMESTAMP, 112) + 
             N'(' + (SELECT COLUMNNAME_ORIGIN FROM @COLUMNNAME_ORIGIN) + N')' + CHAR(13) + CHAR(10) + 
             N'SELECT' + SPACE(1) + (SELECT COLUMNNAME_ORIGIN FROM @COLUMNNAME_ORIGIN) + CHAR(13) + CHAR(10) + 
             N'FROM ' + @TABLENAME_SYNC + N' WITH(NOLOCK)' + CHAR(13) + CHAR(10) + 
             CASE WHEN LEN(@CONDITION_FILTER) > 0 THEN 'WHERE' + SPACE(1) + @CONDITION_FILTER ELSE '' END;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[执行拼接|@CMD_4]
IF @BREAKPOINT >= 10--打印
BEGIN
     PRINT @CMD_4;
END
IF @BREAKPOINT >= 1--执行
BEGIN
     EXEC SP_EXECUTESQL @STMT = @CMD_4;
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[复制索引]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[替换表名|带上库名]
SET @TABLENAME_SAVE = @TABLENAME_DATABASE_ORIGIN + N'.dbo.' + @TABLENAME_TABLE_ORIGIN + N'_SAVE_' + 
                      CONVERT(CHAR(8), CURRENT_TIMESTAMP, 112);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[调用索引复制工具]
EXEC dbo.P_INDEX_CLONE @INDEXFROM_INPUT  = @TABLENAME_ORIGIN,
                       @INDEXSET_INPUT   = @TABLENAME_SAVE,
                       @BREAKPOINT_INPUT = 0;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[交换表名]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[截断替换表名]
SET @TABLENAME_SAVE = REVERSE(SUBSTRING(REVERSE(@TABLENAME_SAVE), 1, CHARINDEX('.', REVERSE(@TABLENAME_SAVE), 1) - 1));
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[动态拼接|交换表名]
SET @CMD_5 = N'USE' + SPACE(1) + QUOTENAME(@TABLENAME_DATABASE_ORIGIN) + CHAR(13) + CHAR(10) + 
             N'IF EXISTS(SELECT TOP 1 * FROM SYSOBJECTS WITH(NOLOCK) WHERE NAME = ''' + @TABLENAME_SAVE + ''')' + CHAR(13) + CHAR(10) + 
             N'BEGIN' + CHAR(13) + CHAR(10) + SPACE(4) + 
             N'IF EXISTS(SELECT TOP 1 * FROM ' + @TABLENAME_SAVE + ' WITH(NOLOCK))' + CHAR(13) + CHAR(10) + SPACE(4) + 
             N'BEGIN' + CHAR(13) + CHAR(10) + SPACE(8) + 
             N'IF OBJECT_ID(''.dbo.' + @TABLENAME_TABLE_ORIGIN + '_STAND_IN'', ''U'') IS NOT NULL DROP TABLE ' + 
             N'.dbo.' + @TABLENAME_TABLE_ORIGIN + N'_STAND_IN' + CHAR(13) + CHAR(10) + SPACE(8) + 
             N'EXEC SP_RENAME ''' + @TABLENAME_TABLE_ORIGIN + ''', ''' + @TABLENAME_TABLE_ORIGIN + '_STAND_IN''' + CHAR(13) + CHAR(10) + SPACE(8) + 
             N'EXEC SP_RENAME ''' + @TABLENAME_SAVE + ''', ''' + @TABLENAME_TABLE_ORIGIN + '''' + CHAR(13) + CHAR(10) + SPACE(4) + 
             N'END' + CHAR(13) + CHAR(10) + 
             N'END'
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[执行拼接|@CMD_5]
IF @BREAKPOINT >= 10--打印
BEGIN
     PRINT @CMD_5;
END
IF @BREAKPOINT >= 1--执行
BEGIN
     EXEC SP_EXECUTESQL @STMT = @CMD_5;
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[预约删除]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[获取替换表名全称|加上服务器名]
SET @TABLENAME_SAVE = @SERVER_NAME + N'.' + @TABLENAME_DATABASE_ORIGIN + N'.dbo.' + @TABLENAME_TABLE_ORIGIN + '_STAND_IN';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[插入预约]
INSERT INTO odsdb.dbo.XTB_SCHEDULED_TABLE_DROP(TABLENAME, CLEANDAY)
SELECT @TABLENAME_SAVE, CONVERT(CHAR(8), DATEADD(DAY, 1, CURRENT_TIMESTAMP), 112);
--*******************************************************************************************************************************************************************
SET NOCOUNT OFF
END
```