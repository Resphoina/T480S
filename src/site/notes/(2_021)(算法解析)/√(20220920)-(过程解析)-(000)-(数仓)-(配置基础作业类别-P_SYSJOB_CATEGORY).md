---
{"dg-publish":true,"permalink":"/2-021/20220920-000-p-sysjob-category/"}
---


# <font color=#DC143C>(20220920)-(过程解析)-(000)-(数仓)-(配置基础作业类别-P_SYSJOB_CATEGORY)</font>
URL:: 

```
dataview
table without id 入榜亮点, 入榜输出
where contains(TITLES, "")
```

```
dataview
table without id 萃取函数, 萃取解法
where contains(TITLES, "")
```

## 查验结果
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[EXEC]
EXEC P_SYSJOB_OTHER_DETAIL;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[READ]
SELECT *
FROM dbo.mrx_sysjobs_info_other WITH(NOLOCK)
WHERE CHARINDEX('徐海权', sysjobs_name, 1) > 0;
```

## (P_SYSJOB_CATEGORY)-(20220920)-(000)-(数仓)
```SQL
CREATE PROC [dbo].[p_sysjob_category]
(@BELONG_TO_WHO_INPUT NVARCHAR(1000))
AS
BEGIN
SET NOCOUNT ON
--*******************************************************************************************************************************************************************
--提出人员:徐海权
--创建人员:徐海权
--创建时间:20220920
--创建用途:配置基础作业类别
--更新频率:一天三次[(00)(徐海权)【仓库管理】【作业监控】作业基础信息扫描(0420|1320|1515)-【RX】]
--*******************************************************************************************************************************************************************
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @BELONG_TO_WHO NVARCHAR(1000);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @BELONG_TO_WHO = N'徐海权';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @BELONG_TO_WHO = @BELONG_TO_WHO_INPUT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CREATE CATEGORY]
IF(SELECT COUNT(*)
   FROM msdb.dbo.syscategories WITH(NOLOCK)
   WHERE category_class = 1
   AND   category_type = 1
   AND   [name] = N'BASIC') < 1-- PRINT 1
EXECUTE msdb.dbo.sp_add_category @class = 'JOB', @type = 'LOCAL', @name = 'BASIC';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[获取基础作业列表]
IF OBJECT_ID('tempdb.dbo.#SYSJOBS_BASIC', 'U') IS NOT NULL DROP TABLE #SYSJOBS_BASIC;
WITH CTE_SYSJOBS_OWNER_OUTPUT
AS (SELECT JOB_ID
    FROM dbo.XTB_SYSJOBS_OWNER_OUTPUT WITH(NOLOCK)
    WHERE JOB_OWNER IN (SELECT VALUE FROM STRING_SPLIT(@BELONG_TO_WHO, '|')))
SELECT JOB_ID, SYSJOBS_NAME, DATABASE_PRINCIPALS_NAME, SYSCATEGORIES_NAME, DESCRIPTION, ENABLED, DATE_CREATED, DATE_MODIFIED,
       SERVERS_NAME, STEP_ID, STEP_NAME, COMMAND, DATABASE_NAME, SCHEDULE_UID, SYSSCHEDULES_SCHEDULE_UID, SYSSCHEDULES_NAME,
       DELETE_LEVEL, ROW_NUMBER()OVER(ORDER BY JOB_ID) AS NUM
INTO #SYSJOBS_BASIC
FROM dbo.mrx_sysjobs_info_other WITH(NOLOCK)/*作业信息补充表*/
WHERE JOB_ID IN (SELECT JOB_ID FROM CTE_SYSJOBS_OWNER_OUTPUT)
AND   SYSCATEGORIES_NAME != 'BASIC';
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[配置作业类别]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @CMD NVARCHAR(MAX);
DECLARE @NUM INT;
DECLARE @JOB_ID_TRAN UNIQUEIDENTIFIER;
DECLARE @CATEGORY_NAME_TRAN NVARCHAR(1000);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @CMD = 'USE MSDB' + CHAR(13) + CHAR(10) + 
           'EXEC dbo.SP_UPDATE_JOB @JOB_ID = @JOB_ID_INSIDE, @CATEGORY_NAME = @CATEGORY_NAME_INSIDE'
SET @NUM = 1;
SET @CATEGORY_NAME_TRAN = N'BASIC';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[循环配置作业类别]
WHILE @NUM <= (SELECT MAX(NUM)FROM #SYSJOBS_BASIC WITH(NOLOCK))
BEGIN
     --∷∷∷∷∷∷[MATCH JOB_ID WITH @NUM]
     SELECT @JOB_ID_TRAN = JOB_ID
     FROM #SYSJOBS_BASIC WITH(NOLOCK)
     WHERE NUM = @NUM;
     --∷∷∷∷∷∷[SP_EXECUTESQL]
     EXEC SP_EXECUTESQL @STMT = @CMD, @PARAMS = N'@JOB_ID_INSIDE UNIQUEIDENTIFIER, @CATEGORY_NAME_INSIDE NVARCHAR(1000)',
                        @JOB_ID_INSIDE = @JOB_ID_TRAN,
                        @CATEGORY_NAME_INSIDE = @CATEGORY_NAME_TRAN
     --∷∷∷∷∷∷[NEXT]
     SET @NUM = @NUM + 1;
END
--*******************************************************************************************************************************************************************
SET NOCOUNT OFF
END
```