---
{"dg-publish":true,"permalink":"/(2_021)(算法解析)/√(20220225)-(过程解析)-(000)-(数仓)-(枪毙进程-P_RUNNING_KILL)/"}
---


# <font color=#DC143C>(20220225)-(过程解析)-(000)-(数仓)-(枪毙进程-P_RUNNING_KILL)</font>
URL:: 

```
dataview
table without id 入榜亮点, 入榜输出
where contains(TITLES, "")
```

| 萃取重点 | 萃取难点 | 萃取锚点 | 萃取输出 |
| ---- | ---- | ---- | ---- |
| \-   | \-   | \-   | \-   |


```toc
```

## 01.逻辑压缩
1. 实时查询所有执行脚本
    1. 调用过程:`P_ALLRUN`
2. 生成临表`#RUNNING`
    1. 源表:`dbo.XTB_ALL_RUN`
    2. 源表:`dbo.sysjob_running_sessionid`
    3. 源表:`dbo.XTB_sysjobs_owner_output`
3. 黑白名单:插入目标`TABLE|dbo.WithoutKill_job`
    1. 白名单:变量传输`@WHITELIST`
        1. 放行白名单增加`102470`<sub><mark style="background: #FF5582A6;">20220921-新增</mark></sub>
    2. 手动插入白名单:`TABLE|SessionID_WithoutKill`
        1. 删除执行完毕的脚本
4. 判断堵塞个数决定删除时间界限
    1. 抓取超过30分钟的脚本进入**_枪毙池_**
    2. 判断步骤[4.1]的个数
        1. 大于等于10个→30分钟以内保留
        2. 大于等于7个→45分钟以内保留
        3. 小于7个→60分钟以内保留
5. 调用过程`[PS_BigBrother]`查询TEMPDB<u>资源占用前2名</u><sub><mark style="background: #FF5582A6;">20220921-迭代-(资源占用前2名→资源占用前3名)</mark> </sub>且超过30分钟的(跳过上一步骤的堵塞个数和堵塞类型的判断)，回收进入**_枪毙池_**
    1. [[(2_021)(算法解析)/√(20220224)-(过程解析)-(000)-(数仓)-(全面活动监控应用-PS_BigBrother)|全面活动监控应用-PS_BigBrother]]
    2. 二选一满足一个即可判断`TEMPDB_ALLOCATIONS`和`TEMPDB_CURRENT`
        1. `(TEMPDB_ALLOCATIONS+TEMPDB_CURRENT) > 2500000`&`TEMPDB_CURRENT >= 500000`
        2. `(TEMPDB_ALLOCATIONS+TEMPDB_CURRENT) > 2500000`&`TEMPDB_ALLOCATIONS >= 500000`
6. 剔除基础作业:`源头#running→生成#overtime`
    1. 赦免基础作业
        1. 判断依据:作业标识是否带数字
        2. 剔除基础作业添加条件限制：作业类别配置=BASIC<sub><mark style="background: #FF5582A6;">20220921-新增</mark> </sub>
    2. 回插作弊作业
        1. 表初始化:`SELECT TOP 0 job_id INTO CHEAT_JOB_ID FROM msdb.dbo.sysjobs WITH(NOLOCK);`
7. 剔除白名单
    1. 目标表`#overtime`删除记录(关联`dbo.WithoutKill_job`)[3]
8. 等待资源类型|WAIT_TYPE
    1. 必杀黑名单→逆向等同赦免特定"等待资源类型"
        1. `WHERE 等待资源类型 IN ('CXPACKET', 'IO_COMPLETION', 'OLEDB', 'PAGEIOLATCH_EX', 'PAGEIOLATCH_SH', 'ASYNC_NETWORK_IO', 'ASYNC_IO_COMPLETION');`
    2. 黑名单范围以外同一类型数量超过3个→回插名单杀死
9. 超过90分钟的脚本回插→无条件杀死<sub><mark style="background: #FF5582A6;">20220921-新增</mark> </sub>
10. 发送通知优化:`源头#RUNNING_KILL→生成#SEND_OA`
11. 操作存档记录
    1. 作业过程
    2. 临时脚本
12. KILL操作执行

## 2.[上传存档|20220920]-[061]-[P_RUNNING_KILL]
```SQL
USE [odsdb]
GO
/****** Object:  StoredProcedure [dbo].[P_RUNNING_KILL]    Script Date: 2022/9/20 11:37:06 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROC [dbo].[P_RUNNING_KILL]
AS
BEGIN
SET NOCOUNT ON;
--*******************************************************************************************************************************************************************
--提出人员:徐海权
--创建人员:叶佩勤
--创建时间:20210830
--创建用途:监控堵塞卡顿脚本
--更新频率:每隔20分钟执行一次
--修改人员:
--报表维度:
--相关过程:P_RUNNING_RECORD(修改前20220208)
--生成结果:SELECT TOP 300 * FROM dbo.KILL_RUNNINGJOB WITH(NOLOCK) ORDER BY DEALTIME DESC
           /*单列不执行白名单(NOT KILL)*/--SELECT * FROM SessionID_WithoutKill WITH(NOLOCK)
--修改人员:徐海权(20220208)
--修改内容:增加防止作弊作业目录、修改判断决定删除时间界限的堵塞个数
           /*SELECT * FROM CHEAT_JOB_ID WITH(NOLOCK);*/
           /*INSERT INTO CHEAT_JOB_ID SELECT '4ED0B360-07D6-4F21-98F5-79956DDBBFE6';*/
--修改人员:徐海权(20220224)
--修改内容:调用过程[PS_BigBrother]把TEMPDB占用最多前2名回收KILL(跳过堵塞个数和堵塞类型的判断)
--修改人员:徐海权(20220225)
--修改内容:TEMPDB_ALLOCATIONS&TEMPDB_CURRENT优化判断限制
--修改人员:徐海权(20220920)
--修改内容:放行白名单增加|资源占用前2名→资源占用前3名|剔除基础作业添加条件限制：作业类别配置=BASIC|超过90分钟的脚本回插→无条件杀死
--*******************************************************************************************************************************************************************
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[赋值变量]
/*DECLARE*/
DECLARE @WHITELIST NVARCHAR(MAX);
DECLARE @ANTI_THEFT INT;
/*SET*/
SET @WHITELIST = N'Administrator|BIGDATA|104282|102470';
SET @ANTI_THEFT = 1;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[获取最新执行情况]
EXEC P_ALLRUN /*@BREAKPOINT_INPUT*/0, /*@SHOW_OR_NOT_INPUT*/0;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[基本信息整理][生成#RUNNING]
IF OBJECT_ID('tempdb.dbo.#running', 'u') IS NOT NULL DROP TABLE #running;
SELECT ROW_NUMBER()OVER(ORDER BY a.开始时间) AS NUM,
       b.JOB_ID,
       b.Job_Name,
       a.loginame,
       CASE WHEN LEN(b.Job_Name) > 0 THEN '执行作业' ELSE '临时脚本' END AS style,
       CASE WHEN a.username IS NULL THEN c.JOB_OWNER ELSE a.username END AS username,
       a.开始时间 AS STARTIME,
       a.session_id,
       a.等待资源类型,
       a.[sql执行语句],
       DATEDIFF(MINUTE, a.开始时间, GETDATE()) * 1.00 / 60 AS 运行时间
INTO #running
FROM dbo.XTB_ALL_RUN                         AS a WITH(NOLOCK)
LEFT OUTER JOIN dbo.sysjob_running_sessionid AS b WITH(NOLOCK)
ON a.SESSION_ID = b.SESSIONID
LEFT OUTER JOIN dbo.XTB_sysjobs_owner_output AS c WITH(NOLOCK)
ON b.JOB_ID = c.JOB_ID;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[名单处理]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[放行白名单(大数据|作业执行|)]
WITH CTE_STRING_SPLIT
AS (SELECT VALUE
    FROM STRING_SPLIT(@WHITELIST, '|'))
    --FROM STRING_SPLIT('Administrator|BIGDATA|104282', '|'))
DELETE a
FROM #running               AS a WITH(NOLOCK)
CROSS JOIN CTE_STRING_SPLIT AS b WITH(NOLOCK)
WHERE CHARINDEX(VALUE, loginame, 1) > 0;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[新增入仓白名单|TABLE|SessionID_WithoutKill]
/*单列不执行白名单(NOT KILL)*/--SELECT * FROM SessionID_WithoutKill WITH(NOLOCK)
INSERT INTO WithoutKill_job(JOB_NAME, STYLE, USERNAME, StarTime, SESSION_ID)
SELECT a.job_name, a.style, a.username, a.StarTime, a.session_id
FROM #running                    AS a WITH(NOLOCK)
INNER JOIN SessionID_WithoutKill AS b WITH(NOLOCK)/*手动插入白名单*/
ON a.SESSION_ID = b.SESSION_ID
EXCEPT
SELECT JOB_NAME, STYLE, USERNAME, StarTime, SESSION_ID FROM WithoutKill_job WITH(NOLOCK);/*剔除上次留存*/
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[删除旧记录]
WITH OLD_RECORD
AS (SELECT JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID
    FROM WithoutKill_job WITH(NOLOCK)
    EXCEPT
    SELECT JOB_NAME, STYLE, USERNAME, STARTIME, a.SESSION_ID
    FROM #running AS a WITH(NOLOCK)
    INNER JOIN SessionID_WithoutKill AS b WITH(NOLOCK)
    ON a.SESSION_ID = b.SESSION_ID)
DELETE a
FROM WithoutKill_job AS a WITH(NOLOCK)
INNER JOIN OLD_RECORD AS b WITH(NOLOCK)
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.StarTime = b.StarTime
AND a.SESSION_ID = b.SESSION_ID
WHERE 1 = 1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[删除执行完毕的脚本|SessionID_WithoutKill]
DELETE FROM SessionID_WithoutKill
WHERE SESSION_ID NOT IN (SELECT SESSION_ID FROM WithoutKill_job);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[判断堵塞个数决定删除时间界限]
--①抓取超过30分钟的脚本
--②判断步骤①的个数
----大于等于10个→30分钟以内保留
----大于等于7个→45分钟以内保留
----小于7个→60分钟以内保留
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @COUNT INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SELECT @COUNT = COUNT(0)
FROM #running WITH(NOLOCK)
WHERE [运行时间] > 0.5;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[资源占用判断]
/*EXEC PROC*/
EXEC PS_BigBrother @MINUTE_FROM_NOW = 1, @QUICKSHOW = 1;/*调取此刻执行结果*/
/*TEMPDB占用最多且运行时间超过30分钟*/
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL_TEMPDB', 'U') IS NOT NULL DROP TABLE #RUNNING_KILL_TEMPDB;
WITH CTE_TEMPDB_ALLOCATIONS
AS (SELECT ROW_NUMBER()OVER(ORDER BY TEMPDB_ALLOCATIONS DESC) AS TEMPDB_ALLOCATIONS_DESC,
           START_TIME,
           LOGIN_TIME,
           RUN_DURATION,
           SESSION_ID
    FROM XTB_BIGBROTHER_LAST WITH(NOLOCK)
    /*20220225-徐海权-新增限定*/
    WHERE CAST(LTRIM(REPLACE(TEMPDB_ALLOCATIONS, ',', '')) AS INT) >= 500000
    AND (CAST(LTRIM(REPLACE(TEMPDB_ALLOCATIONS, ',', '')) AS INT) + CAST(LTRIM(REPLACE(TEMPDB_CURRENT, ',', '')) AS INT)) > 2500000),
     CTE_TEMPDB_CURRENT
AS (SELECT ROW_NUMBER()OVER(ORDER BY TEMPDB_CURRENT DESC) AS TEMPDB_CURRENT_DESC,
           START_TIME,
           LOGIN_TIME,
           RUN_DURATION,
           SESSION_ID
    FROM XTB_BIGBROTHER_LAST WITH(NOLOCK)
    /*20220225-徐海权-新增限定*/
    WHERE CAST(LTRIM(REPLACE(TEMPDB_CURRENT, ',', '')) AS INT) >= 500000
    AND (CAST(LTRIM(REPLACE(TEMPDB_ALLOCATIONS, ',', '')) AS INT) + CAST(LTRIM(REPLACE(TEMPDB_CURRENT, ',', '')) AS INT)) > 2500000),
     CTE_TEMPDB_RESULT
AS (SELECT TEMPDB_ALLOCATIONS_DESC, START_TIME, LOGIN_TIME, RUN_DURATION, SESSION_ID
    FROM CTE_TEMPDB_ALLOCATIONS WITH(NOLOCK)
    WHERE TEMPDB_ALLOCATIONS_DESC < 4
    UNION ALL
    SELECT TEMPDB_CURRENT_DESC, START_TIME, LOGIN_TIME, RUN_DURATION, SESSION_ID
    FROM CTE_TEMPDB_CURRENT WITH(NOLOCK)
    WHERE TEMPDB_CURRENT_DESC < 4)
--SELECT * FROM CTE_TEMPDB_RESULT WITH(NOLOCK)
SELECT LEFT(JOB_NAME, 4) AS JOB_NAME_BASIC,
       JOB_NAME,
       JOB_ID,
       STYLE,
       USERNAME,
       STARTIME,
       SESSION_ID,
       [等待资源类型],
       [sql执行语句],
       [运行时间],
       CURRENT_TIMESTAMP AS DEALTIME
INTO #RUNNING_KILL_TEMPDB
FROM #RUNNING WITH(NOLOCK)
WHERE SESSION_ID IN(SELECT SESSION_ID FROM CTE_TEMPDB_RESULT WITH(NOLOCK))
AND [运行时间] > 0.5;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[超时判断|TIMEIF→DELETE]
IF @COUNT >= 10
BEGIN
     DELETE FROM #running WHERE [运行时间] < 0.5;/*@COUNT >= 10→过滤小于30分钟*/
END;
ELSE IF @COUNT >= 7
BEGIN
     DELETE FROM #running WHERE [运行时间] < 0.75;/*@COUNT >= 7→过滤小于45分钟*/
END;
ELSE IF @COUNT < 7
BEGIN
     DELETE FROM #running WHERE [运行时间] < 1.00;/*@COUNT < 7→过滤小于60分钟*/
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[剔除基础作业][源头#running→生成#overtime]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CREATE|RESET]
--SELECT TOP 0 job_id INTO CHEAT_JOB_ID FROM msdb.dbo.sysjobs WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[赦免基础作业]
--判断依据:作业标识是否带数字
IF OBJECT_ID('tempdb.dbo.#overtime', 'u') IS NOT NULL DROP TABLE #overtime;
WITH JOB_NAME_CATCH
AS(SELECT LEFT(JOB_NAME, 4) AS JOB_NAME_BASIC,
          JOB_NAME,
          JOB_ID,
          STYLE,
          USERNAME,
          STARTIME,
          SESSION_ID,
          [等待资源类型],
          [sql执行语句],
          [运行时间],
          CURRENT_TIMESTAMP AS DEALTIME
FROM #running WITH(NOLOCK)),
SCHEDULE_UID_ROW_NUMBER
AS (SELECT job_id, schedule_uid, syscategories_name,
    ROW_NUMBER()OVER(PARTITION BY JOB_ID ORDER BY SCHEDULE_UID) AS NUM
    FROM dbo.mrx_sysjobs_info_other WITH(NOLOCK)),
SCHEDULE_UID_ROW_NUMBER_ONE
AS (SELECT job_id, schedule_uid, syscategories_name
    FROM SCHEDULE_UID_ROW_NUMBER WITH(NOLOCK)
    WHERE num = 1)
SELECT a.JOB_NAME_BASIC,
       a.JOB_NAME,
       a.JOB_ID,
       a.STYLE,
       a.USERNAME,
       a.STARTIME,
       a.SESSION_ID,
       a.[等待资源类型],
       a.[sql执行语句],
       a.[运行时间],
       a.DEALTIME,
       b.syscategories_name
INTO #overtime
FROM JOB_NAME_CATCH AS a WITH(NOLOCK)
LEFT OUTER JOIN SCHEDULE_UID_ROW_NUMBER_ONE AS b WITH(NOLOCK)
ON a.JOB_ID = b.JOB_ID
WHERE ISNULL(JOB_NAME_BASIC, 'NULL_INPUT') NOT LIKE '%[0-9]%'
OR    ISNULL(b.syscategories_name, 'NULL_INPUT') != 'BASIC';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[回插作弊作业]
IF @ANTI_THEFT = 1
BEGIN
     INSERT INTO #overtime(JOB_NAME_BASIC, JOB_NAME, JOB_ID, style, username, STARTIME, session_id, 等待资源类型, sql执行语句, 运行时间, DEALTIME)
     SELECT LEFT(a.JOB_NAME, 4) AS JOB_NAME_BASIC,
            a.JOB_NAME,
            a.JOB_ID,
            a.style,
            a.username,
            a.STARTIME,
            a.session_id,
            a.[等待资源类型],
            a.[sql执行语句],
            a.[运行时间],
            CURRENT_TIMESTAMP AS DEALTIME
     FROM #running               AS a WITH(NOLOCK)
     INNER JOIN dbo.CHEAT_JOB_ID AS b WITH(NOLOCK)
     ON a.JOB_ID = b.JOB_ID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[剔除白名单]
DELETE a
FROM #overtime             AS a WITH(NOLOCK)
INNER JOIN WithoutKill_job AS b WITH(NOLOCK)
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.STARTIME = b.STARTIME
AND a.SESSION_ID = b.SESSION_ID;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[等待资源类型|WAIT_TYPE][源头#OVERTIME→生成#RUNNING_KILL]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[必杀黑名单]
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL', 'u') IS NOT NULL DROP TABLE #RUNNING_KILL;
SELECT JOB_NAME_BASIC,
       JOB_NAME,
       JOB_ID,
       STYLE,
       USERNAME,
       STARTIME,
       SESSION_ID,
       等待资源类型,
       SQL执行语句,
       运行时间,
       DEALTIME,
       '超时终止' AS KILL_STYLE
INTO #RUNNING_KILL
FROM #OVERTIME WITH(NOLOCK)
WHERE 等待资源类型 IN ('CXPACKET', 'IO_COMPLETION', 'OLEDB', 'PAGEIOLATCH_EX', 'PAGEIOLATCH_SH', 'ASYNC_NETWORK_IO', 'ASYNC_IO_COMPLETION');
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[黑名单范围以外数量超过3个→回插名单杀死]
WITH CTE_COUNT
AS (SELECT [等待资源类型], COUNT(0) AS CNT
    FROM #OVERTIME WITH(NOLOCK)
    GROUP BY [等待资源类型]
    HAVING COUNT(0) > 3)
INSERT INTO #RUNNING_KILL(JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME, KILL_STYLE)
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID,
       a.[等待资源类型], a.[SQL执行语句], a.[运行时间], a.DEALTIME, '超时终止' AS KILL_STYLE
FROM #overtime       AS a WITH(NOLOCK)
INNER JOIN CTE_COUNT AS b WITH(NOLOCK)
ON a.[等待资源类型] = b.[等待资源类型]
EXCEPT
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME, KILL_STYLE
FROM #RUNNING_KILL;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[TEMPDB占用最多回插]
INSERT INTO #RUNNING_KILL(JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME, KILL_STYLE)
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME, '资源终止' AS KILL_STYLE
FROM #RUNNING_KILL_TEMPDB WITH(NOLOCK)
EXCEPT
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME, KILL_STYLE
FROM #RUNNING_KILL WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[20220920-超过90分钟回插]
INSERT INTO #RUNNING_KILL(JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME, KILL_STYLE)
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME, '超时终止' AS KILL_STYLE
FROM #RUNNING_KILL_TEMPDB WITH(NOLOCK)
WHERE [运行时间] >= 1.5
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[通知优化][源头#RUNNING_KILL→生成#SEND_OA]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[生成内容]
IF OBJECT_ID('tempdb.dbo.#send_oa', 'u') IS NOT NULL DROP TABLE #send_oa;
SELECT IDENTITY(INT, 1, 1) AS ID,
       a.JOB_NAME,
       a.STYLE,
       a.USERNAME,
       a.STARTIME,
       a.SESSION_ID,
       a.[等待资源类型],
       a.USERNAME + ':通知！[SESSION_ID]号:' + CONVERT(VARCHAR(30), SESSION_ID) + '的[061]脚本超时堵塞已被KILL，麻烦优化!' + [SQL执行语句] AS NR,
       a.[运行时间],
       a.DEALTIME,
       b.USERID,
       b.PHONE
INTO #send_oa
FROM #RUNNING_KILL                 AS a WITH(NOLOCK)
INNER JOIN odsdbbi.dbo.MemberForBI AS b WITH(NOLOCK)
ON a.username = b.username;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[发送通知]
--❖❖❖❖❖❖[DECLARE]
DECLARE @CONTENT VARCHAR(MAX);
DECLARE @USERID VARCHAR(200);
DECLARE @ID INT;
--❖❖❖❖❖❖[WHILE]
WHILE EXISTS(SELECT TOP 1 * FROM #send_oa WITH(NOLOCK))
BEGIN
     SET @ID = (SELECT TOP 1 ID FROM #send_oa ORDER BY ID);
     SET @CONTENT = (SELECT NR FROM #send_oa WHERE @ID = ID);
     SET @USERID = (SELECT USERID FROM #send_oa WHERE @ID = ID);
     /*NEXT*/
     DELETE FROM #send_oa WHERE ID = @ID;
     /*SEND*/
     EXEC DATABASE_13.C6.dbo.SP_SendMessageToOAUser_Proc @CONTENT, @USERID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[操作存档记录]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DELETE|作业过程]
DELETE a
FROM dbo.KILL_RUNNINGJOB AS a
INNER JOIN #RUNNING_KILL AS b
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.SESSION_ID = b.SESSION_ID
WHERE a.HOSTNAME IS NULL/*[p_KILL_BEFORE_RESTART]结果字段HOSTNAME不为空*/
AND   b.等待资源类型 IS NOT NULL;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DELETE|临时脚本]
DELETE a
FROM dbo.KILL_RUNNINGJOB AS a
INNER JOIN #RUNNING_KILL AS b
ON ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.SESSION_ID = b.SESSION_ID
WHERE a.HOSTNAME IS NULL
AND   b.等待资源类型 IS NOT NULL
AND   b.STYLE = '临时脚本';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[INSERT]
INSERT INTO dbo.KILL_RUNNINGJOB(JOB_NAME, STYLE, USERNAME, 开始时间, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME, KILL_STYLE)
SELECT JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID,
       [等待资源类型], 
       LEFT([SQL执行语句],100) AS [SQL执行语句],
       [运行时间],
       CURRENT_TIMESTAMP AS DEALTIME,
       KILL_STYLE
FROM #RUNNING_KILL
WHERE 等待资源类型 IS NOT NULL;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[KILL操作执行]
/*DECLARE*/
DECLARE @CMD NVARCHAR(MAX);
DECLARE @SESSION_ID NVARCHAR(100);
/*WHILE*/
WHILE EXISTS (SELECT TOP 1 * FROM #RUNNING_KILL WITH(NOLOCK))
BEGIN
     SELECT TOP 1 @SESSION_ID = SESSION_ID FROM #RUNNING_KILL WITH(NOLOCK);
     SET @CMD = N'KILL ' + @SESSION_ID;
     EXEC SP_EXECUTESQL @STMT = @CMD;
     DELETE FROM #RUNNING_KILL WHERE SESSION_ID = @SESSION_ID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[删除临表]
IF OBJECT_ID('tempdb.dbo.#running', 'u') IS NOT NULL DROP TABLE #running;
IF OBJECT_ID('tempdb.dbo.#overtime', 'u') IS NOT NULL DROP TABLE #overtime;
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL', 'u') IS NOT NULL DROP TABLE #RUNNING_KILL;
--*******************************************************************************************************************************************************************
SET NOCOUNT OFF
END
```

## 2.[上传存档|20220225]-[061]-[P_RUNNING_KILL]
```SQL
USE [odsdb]
GO
/****** Object:  StoredProcedure [dbo].[P_RUNNING_KILL]    Script Date: 2022/2/24 10:42:24 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROC [dbo].[P_RUNNING_KILL]
AS
BEGIN
SET NOCOUNT ON;
--*******************************************************************************************************************************************************************
--提出人员:徐海权
--创建人员:叶佩勤
--创建时间:20210830
--创建用途:监控堵塞卡顿脚本
--更新频率:每隔20分钟执行一次
--修改人员:
--报表维度:
--相关过程:P_RUNNING_RECORD(修改前20220208)
--生成结果:SELECT TOP 300 * FROM dbo.KILL_RUNNINGJOB WITH(NOLOCK) ORDER BY DEALTIME DESC
           /*单列不执行白名单(NOT KILL)*/--SELECT * FROM SessionID_WithoutKill WITH(NOLOCK)
--修改人员:徐海权(20220208)
--修改内容:增加防止作弊作业目录、修改判断决定删除时间界限的堵塞个数
           /*SELECT * FROM CHEAT_JOB_ID WITH(NOLOCK);*/
           /*INSERT INTO CHEAT_JOB_ID SELECT '4ED0B360-07D6-4F21-98F5-79956DDBBFE6';*/
--修改人员:徐海权(20220224)
--修改内容:调用过程[PS_BigBrother]把TEMPDB占用最多前2名回收KILL(跳过堵塞个数和堵塞类型的判断)
--修改人员:徐海权(20220225)
--修改内容:TEMPDB_ALLOCATIONS&TEMPDB_CURRENT优化判断限制
--*******************************************************************************************************************************************************************
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[赋值变量]
/*DECLARE*/
DECLARE @WHITELIST NVARCHAR(MAX);
DECLARE @ANTI_THEFT INT;
/*SET*/
SET @WHITELIST = N'Administrator|BIGDATA|104282';
SET @ANTI_THEFT = 1;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[获取最新执行情况]
EXEC P_ALLRUN /*@BREAKPOINT_INPUT*/0, /*@SHOW_OR_NOT_INPUT*/0;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[基本信息整理][生成#RUNNING]
IF OBJECT_ID('tempdb.dbo.#running', 'u') IS NOT NULL DROP TABLE #running;
SELECT ROW_NUMBER()OVER(ORDER BY a.开始时间) AS NUM,
       b.JOB_ID,
       b.Job_Name,
       a.loginame,
       CASE WHEN LEN(b.Job_Name) > 0 THEN '执行作业' ELSE '临时脚本' END AS style,
       CASE WHEN a.username IS NULL THEN c.JOB_OWNER ELSE a.username END AS username,
       a.开始时间 AS STARTIME,
       a.session_id,
       a.等待资源类型,
       a.[sql执行语句],
       DATEDIFF(MINUTE, a.开始时间, GETDATE()) * 1.00 / 60 AS 运行时间
INTO #running
FROM dbo.XTB_ALL_RUN                         AS a WITH(NOLOCK)
LEFT OUTER JOIN dbo.sysjob_running_sessionid AS b WITH(NOLOCK)
ON a.SESSION_ID = b.SESSIONID
LEFT OUTER JOIN dbo.XTB_sysjobs_owner_output AS c WITH(NOLOCK)
ON b.JOB_ID = c.JOB_ID;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[名单处理]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[放行白名单(大数据|作业执行|)]
WITH CTE_STRING_SPLIT
AS (SELECT VALUE
    FROM STRING_SPLIT(@WHITELIST, '|'))
    --FROM STRING_SPLIT('Administrator|BIGDATA|104282', '|'))
DELETE a
FROM #running               AS a WITH(NOLOCK)
CROSS JOIN CTE_STRING_SPLIT AS b WITH(NOLOCK)
WHERE CHARINDEX(VALUE, loginame, 1) > 0;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[新增入仓白名单|TABLE|SessionID_WithoutKill]
/*单列不执行白名单(NOT KILL)*/--SELECT * FROM SessionID_WithoutKill WITH(NOLOCK)
INSERT INTO WithoutKill_job(JOB_NAME, STYLE, USERNAME, StarTime, SESSION_ID)
SELECT a.job_name, a.style, a.username, a.StarTime, a.session_id
FROM #running                    AS a WITH(NOLOCK)
INNER JOIN SessionID_WithoutKill AS b WITH(NOLOCK)/*手动插入白名单*/
ON a.SESSION_ID = b.SESSION_ID
EXCEPT
SELECT JOB_NAME, STYLE, USERNAME, StarTime, SESSION_ID FROM WithoutKill_job WITH(NOLOCK);/*剔除上次留存*/
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[删除旧记录]
WITH OLD_RECORD
AS (SELECT JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID
    FROM WithoutKill_job WITH(NOLOCK)
    EXCEPT
    SELECT JOB_NAME, STYLE, USERNAME, STARTIME, a.SESSION_ID
    FROM #running AS a WITH(NOLOCK)
    INNER JOIN SessionID_WithoutKill AS b WITH(NOLOCK)
    ON a.SESSION_ID = b.SESSION_ID)
DELETE a
FROM WithoutKill_job AS a WITH(NOLOCK)
INNER JOIN OLD_RECORD AS b WITH(NOLOCK)
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.StarTime = b.StarTime
AND a.SESSION_ID = b.SESSION_ID
WHERE 1 = 1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[删除执行完毕的脚本|SessionID_WithoutKill]
DELETE FROM SessionID_WithoutKill
WHERE SESSION_ID NOT IN (SELECT SESSION_ID FROM WithoutKill_job);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[判断堵塞个数决定删除时间界限]
--①抓取超过30分钟的脚本
--②判断步骤①的个数
----大于等于10个→30分钟以内保留
----大于等于7个→45分钟以内保留
----小于7个→60分钟以内保留
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @COUNT INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SELECT @COUNT = COUNT(0)
FROM #running WITH(NOLOCK)
WHERE [运行时间] > 0.5;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[资源占用判断]
/*EXEC PROC*/
EXEC PS_BigBrother @MINUTE_FROM_NOW = 1, @QUICKSHOW = 1;/*调取此刻执行结果*/
/*TEMPDB占用最多且运行时间超过30分钟*/
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL_TEMPDB', 'U') IS NOT NULL DROP TABLE #RUNNING_KILL_TEMPDB;
WITH CTE_TEMPDB_ALLOCATIONS
AS (SELECT ROW_NUMBER()OVER(ORDER BY TEMPDB_ALLOCATIONS DESC) AS TEMPDB_ALLOCATIONS_DESC,
           START_TIME,
           LOGIN_TIME,
           RUN_DURATION,
           SESSION_ID
    FROM XTB_BIGBROTHER_LAST WITH(NOLOCK)
    /*20220225-徐海权-新增限定*/
    WHERE CAST(LTRIM(REPLACE(TEMPDB_ALLOCATIONS, ',', '')) AS INT) >= 500000
    AND (CAST(LTRIM(REPLACE(TEMPDB_ALLOCATIONS, ',', '')) AS INT) + CAST(LTRIM(REPLACE(TEMPDB_CURRENT, ',', '')) AS INT)) > 2500000),
     CTE_TEMPDB_CURRENT
AS (SELECT ROW_NUMBER()OVER(ORDER BY TEMPDB_CURRENT DESC) AS TEMPDB_CURRENT_DESC,
           START_TIME,
           LOGIN_TIME,
           RUN_DURATION,
           SESSION_ID
    FROM XTB_BIGBROTHER_LAST WITH(NOLOCK)
    /*20220225-徐海权-新增限定*/
    WHERE CAST(LTRIM(REPLACE(TEMPDB_CURRENT, ',', '')) AS INT) >= 500000
    AND (CAST(LTRIM(REPLACE(TEMPDB_ALLOCATIONS, ',', '')) AS INT) + CAST(LTRIM(REPLACE(TEMPDB_CURRENT, ',', '')) AS INT)) > 2500000),
     CTE_TEMPDB_RESULT
AS (SELECT TEMPDB_ALLOCATIONS_DESC, START_TIME, LOGIN_TIME, RUN_DURATION, SESSION_ID
    FROM CTE_TEMPDB_ALLOCATIONS WITH(NOLOCK)
    WHERE TEMPDB_ALLOCATIONS_DESC < 3
    UNION ALL
    SELECT TEMPDB_CURRENT_DESC, START_TIME, LOGIN_TIME, RUN_DURATION, SESSION_ID
    FROM CTE_TEMPDB_CURRENT WITH(NOLOCK)
    WHERE TEMPDB_CURRENT_DESC < 3)
--SELECT * FROM CTE_TEMPDB_RESULT WITH(NOLOCK)
SELECT LEFT(JOB_NAME, 4) AS JOB_NAME_BASIC,
       JOB_NAME,
       JOB_ID,
       STYLE,
       USERNAME,
       STARTIME,
       SESSION_ID,
       [等待资源类型],
       [sql执行语句],
       [运行时间],
       CURRENT_TIMESTAMP AS DEALTIME
INTO #RUNNING_KILL_TEMPDB
FROM #RUNNING WITH(NOLOCK)
WHERE SESSION_ID IN(SELECT SESSION_ID FROM CTE_TEMPDB_RESULT WITH(NOLOCK))
AND [运行时间] > 0.5;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[超时判断|TIMEIF→DELETE]
IF @COUNT >= 10
BEGIN
     DELETE FROM #running WHERE [运行时间] < 0.5;/*@COUNT >= 10→过滤小于30分钟*/
END;
ELSE IF @COUNT >= 7
BEGIN
     DELETE FROM #running WHERE [运行时间] < 0.75;/*@COUNT >= 7→过滤小于45分钟*/
END;
ELSE IF @COUNT < 7
BEGIN
     DELETE FROM #running WHERE [运行时间] < 1.00;/*@COUNT < 7→过滤小于60分钟*/
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[剔除基础作业][源头#running→生成#overtime]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CREATE|RESET]
--SELECT TOP 0 job_id INTO CHEAT_JOB_ID FROM msdb.dbo.sysjobs WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[赦免基础作业]
--判断依据:作业标识是否带数字
IF OBJECT_ID('tempdb.dbo.#overtime', 'u') IS NOT NULL DROP TABLE #overtime;
WITH JOB_NAME_CATCH
AS(SELECT LEFT(JOB_NAME, 4) AS JOB_NAME_BASIC,
          JOB_NAME,
          JOB_ID,
          STYLE,
          USERNAME,
          STARTIME,
          SESSION_ID,
          [等待资源类型],
          [sql执行语句],
          [运行时间],
          CURRENT_TIMESTAMP AS DEALTIME
FROM #running WITH(NOLOCK))
SELECT a.JOB_NAME_BASIC,
       a.JOB_NAME,
       a.JOB_ID,
       a.STYLE,
       a.USERNAME,
       a.STARTIME,
       a.SESSION_ID,
       a.[等待资源类型],
       a.[sql执行语句],
       a.[运行时间],
       a.DEALTIME
INTO #overtime
FROM JOB_NAME_CATCH AS a WITH(NOLOCK)
WHERE ISNULL(JOB_NAME_BASIC, 'NULL_INPUT') NOT LIKE '%[0-9]%';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[回插作弊作业]
IF @ANTI_THEFT = 1
BEGIN
     INSERT INTO #overtime(JOB_NAME_BASIC, JOB_NAME, JOB_ID, style, username, STARTIME, session_id, 等待资源类型, sql执行语句, 运行时间, DEALTIME)
     SELECT LEFT(a.JOB_NAME, 4) AS JOB_NAME_BASIC,
            a.JOB_NAME,
            a.JOB_ID,
            a.style,
            a.username,
            a.STARTIME,
            a.session_id,
            a.[等待资源类型],
            a.[sql执行语句],
            a.[运行时间],
            CURRENT_TIMESTAMP AS DEALTIME
     FROM #running               AS a WITH(NOLOCK)
     INNER JOIN dbo.CHEAT_JOB_ID AS b WITH(NOLOCK)
     ON a.JOB_ID = b.JOB_ID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[剔除白名单]
DELETE a
FROM #overtime             AS a WITH(NOLOCK)
INNER JOIN WithoutKill_job AS b WITH(NOLOCK)
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.STARTIME = b.STARTIME
AND a.SESSION_ID = b.SESSION_ID;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[等待资源类型|WAIT_TYPE][源头#OVERTIME→生成#RUNNING_KILL]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[必杀黑名单]
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL', 'u') IS NOT NULL DROP TABLE #RUNNING_KILL;
SELECT JOB_NAME_BASIC,
       JOB_NAME,
       JOB_ID,
       STYLE,
       USERNAME,
       STARTIME,
       SESSION_ID,
       等待资源类型,
       SQL执行语句,
       运行时间,
       DEALTIME,
       '超时终止' AS KILL_STYLE
INTO #RUNNING_KILL
FROM #OVERTIME WITH(NOLOCK)
WHERE 等待资源类型 IN ('CXPACKET', 'IO_COMPLETION', 'OLEDB', 'PAGEIOLATCH_EX', 'PAGEIOLATCH_SH', 'ASYNC_NETWORK_IO', 'ASYNC_IO_COMPLETION');
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[黑名单范围以外数量超过3个→回插名单杀死]
WITH CTE_COUNT
AS (SELECT [等待资源类型], COUNT(0) AS CNT
    FROM #OVERTIME WITH(NOLOCK)
    GROUP BY [等待资源类型]
    HAVING COUNT(0) > 3)
INSERT INTO #RUNNING_KILL(JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME, KILL_STYLE)
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID,
       a.[等待资源类型], a.[SQL执行语句], a.[运行时间], a.DEALTIME, '超时终止' AS KILL_STYLE
FROM #overtime       AS a WITH(NOLOCK)
INNER JOIN CTE_COUNT AS b WITH(NOLOCK)
ON a.[等待资源类型] = b.[等待资源类型]
EXCEPT
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME, KILL_STYLE
FROM #RUNNING_KILL;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[TEMPDB占用最多回插]
INSERT INTO #RUNNING_KILL(JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME, KILL_STYLE)
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME, '资源终止' AS KILL_STYLE
FROM #RUNNING_KILL_TEMPDB WITH(NOLOCK)
EXCEPT
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME, KILL_STYLE
FROM #RUNNING_KILL WITH(NOLOCK);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[通知优化][源头#RUNNING_KILL→生成#SEND_OA]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[生成内容]
IF OBJECT_ID('tempdb.dbo.#send_oa', 'u') IS NOT NULL DROP TABLE #send_oa;
SELECT IDENTITY(INT, 1, 1) AS ID,
       a.JOB_NAME,
       a.STYLE,
       a.USERNAME,
       a.STARTIME,
       a.SESSION_ID,
       a.[等待资源类型],
       a.USERNAME + ':通知！[SESSION_ID]号:' + CONVERT(VARCHAR(30), SESSION_ID) + '的[061]脚本超时堵塞已被KILL，麻烦优化!' + [SQL执行语句] AS NR,
       a.[运行时间],
       a.DEALTIME,
       b.USERID,
       b.PHONE
INTO #send_oa
FROM #RUNNING_KILL                 AS a WITH(NOLOCK)
INNER JOIN odsdbbi.dbo.MemberForBI AS b WITH(NOLOCK)
ON a.username = b.username;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[发送通知]
--❖❖❖❖❖❖[DECLARE]
DECLARE @CONTENT VARCHAR(MAX);
DECLARE @USERID VARCHAR(200);
DECLARE @ID INT;
--❖❖❖❖❖❖[WHILE]
WHILE EXISTS(SELECT TOP 1 * FROM #send_oa WITH(NOLOCK))
BEGIN
     SET @ID = (SELECT TOP 1 ID FROM #send_oa ORDER BY ID);
     SET @CONTENT = (SELECT NR FROM #send_oa WHERE @ID = ID);
     SET @USERID = (SELECT USERID FROM #send_oa WHERE @ID = ID);
     /*NEXT*/
     DELETE FROM #send_oa WHERE ID = @ID;
     /*SEND*/
     EXEC DATABASE_13.C6.dbo.SP_SendMessageToOAUser_Proc @CONTENT, @USERID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[操作存档记录]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DELETE|作业过程]
DELETE a
FROM dbo.KILL_RUNNINGJOB AS a
INNER JOIN #RUNNING_KILL AS b
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.SESSION_ID = b.SESSION_ID
WHERE a.HOSTNAME IS NULL/*[p_KILL_BEFORE_RESTART]结果字段HOSTNAME不为空*/
AND   b.等待资源类型 IS NOT NULL;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DELETE|临时脚本]
DELETE a
FROM dbo.KILL_RUNNINGJOB AS a
INNER JOIN #RUNNING_KILL AS b
ON ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.SESSION_ID = b.SESSION_ID
WHERE a.HOSTNAME IS NULL
AND   b.等待资源类型 IS NOT NULL
AND   b.STYLE = '临时脚本';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[INSERT]
INSERT INTO dbo.KILL_RUNNINGJOB(JOB_NAME, STYLE, USERNAME, 开始时间, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME, KILL_STYLE)
SELECT JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID,
       [等待资源类型], 
       LEFT([SQL执行语句],100) AS [SQL执行语句],
       [运行时间],
       CURRENT_TIMESTAMP AS DEALTIME,
       KILL_STYLE
FROM #RUNNING_KILL
WHERE 等待资源类型 IS NOT NULL;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[KILL操作执行]
/*DECLARE*/
DECLARE @CMD NVARCHAR(MAX);
DECLARE @SESSION_ID NVARCHAR(100);
/*WHILE*/
WHILE EXISTS (SELECT TOP 1 * FROM #RUNNING_KILL WITH(NOLOCK))
BEGIN
     SELECT TOP 1 @SESSION_ID = SESSION_ID FROM #RUNNING_KILL WITH(NOLOCK);
     SET @CMD = N'KILL ' + @SESSION_ID;
     EXEC SP_EXECUTESQL @STMT = @CMD;
     DELETE FROM #RUNNING_KILL WHERE SESSION_ID = @SESSION_ID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[删除临表]
IF OBJECT_ID('tempdb.dbo.#running', 'u') IS NOT NULL DROP TABLE #running;
IF OBJECT_ID('tempdb.dbo.#overtime', 'u') IS NOT NULL DROP TABLE #overtime;
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL', 'u') IS NOT NULL DROP TABLE #RUNNING_KILL;
--*******************************************************************************************************************************************************************
SET NOCOUNT OFF
END
```

## 2.[上传存档|20220224]-[061]-[P_RUNNING_KILL]
```SQL
USE [odsdb]
GO
/****** Object:  StoredProcedure [dbo].[P_RUNNING_KILL]    Script Date: 2022/2/24 10:42:24 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROC [dbo].[P_RUNNING_KILL]
AS
BEGIN
SET NOCOUNT ON;
--*******************************************************************************************************************************************************************
--提出人员:徐海权
--创建人员:叶佩勤
--创建时间:20210830
--创建用途:监控堵塞卡顿脚本
--更新频率:每隔20分钟执行一次
--修改人员:
--报表维度:
--相关过程:P_RUNNING_RECORD(修改前20220208)
--生成结果:SELECT * FROM dbo.KILL_RUNNINGJOB WITH(NOLOCK) ORDER BY DEALTIME DESC
           /*单列不执行白名单(NOT KILL)*/--SELECT * FROM SessionID_WithoutKill WITH(NOLOCK)
--修改人员:徐海权(20220208)
--修改内容:增加防止作弊作业目录、修改判断决定删除时间界限的堵塞个数
           /*SELECT * FROM CHEAT_JOB_ID WITH(NOLOCK);*/
           /*INSERT INTO CHEAT_JOB_ID SELECT '4ED0B360-07D6-4F21-98F5-79956DDBBFE6';*/
--修改人员:徐海权(20220224)
--修改内容:调用过程[PS_BigBrother]把TEMPDB占用最多前2名回收KILL(跳过堵塞个数和堵塞类型的判断)
--*******************************************************************************************************************************************************************
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[赋值变量]
/*DECLARE*/
DECLARE @WHITELIST NVARCHAR(MAX);
DECLARE @ANTI_THEFT INT;
/*SET*/
SET @WHITELIST = N'Administrator|BIGDATA|104282';
SET @ANTI_THEFT = 1;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[获取最新执行情况]
EXEC P_ALLRUN /*@BREAKPOINT_INPUT*/0, /*@SHOW_OR_NOT_INPUT*/0;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[基本信息整理][生成#RUNNING]
IF OBJECT_ID('tempdb.dbo.#running', 'u') IS NOT NULL DROP TABLE #running;
SELECT ROW_NUMBER()OVER(ORDER BY a.开始时间) AS NUM,
       b.JOB_ID,
       b.Job_Name,
       a.loginame,
       CASE WHEN LEN(b.Job_Name) > 0 THEN '执行作业' ELSE '临时脚本' END AS style,
       CASE WHEN a.username IS NULL THEN c.JOB_OWNER ELSE a.username END AS username,
       a.开始时间 AS STARTIME,
       a.session_id,
       a.等待资源类型,
       a.[sql执行语句],
       DATEDIFF(MINUTE, a.开始时间, GETDATE()) * 1.00 / 60 AS 运行时间
INTO #running
FROM dbo.XTB_ALL_RUN                         AS a WITH(NOLOCK)
LEFT OUTER JOIN dbo.sysjob_running_sessionid AS b WITH(NOLOCK)
ON a.SESSION_ID = b.SESSIONID
LEFT OUTER JOIN dbo.XTB_sysjobs_owner_output AS c WITH(NOLOCK)
ON b.JOB_ID = c.JOB_ID;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[名单处理]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[放行白名单(大数据|作业执行|)]
WITH CTE_STRING_SPLIT
AS (SELECT VALUE
    FROM STRING_SPLIT(@WHITELIST, '|'))
    --FROM STRING_SPLIT('Administrator|BIGDATA|104282', '|'))
DELETE a
FROM #running               AS a WITH(NOLOCK)
CROSS JOIN CTE_STRING_SPLIT AS b WITH(NOLOCK)
WHERE CHARINDEX(VALUE, loginame, 1) > 0;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[新增入仓白名单|TABLE|SessionID_WithoutKill]
/*单列不执行白名单(NOT KILL)*/--SELECT * FROM SessionID_WithoutKill WITH(NOLOCK)
INSERT INTO WithoutKill_job(JOB_NAME, STYLE, USERNAME, StarTime, SESSION_ID)
SELECT a.job_name, a.style, a.username, a.StarTime, a.session_id
FROM #running                    AS a WITH(NOLOCK)
INNER JOIN SessionID_WithoutKill AS b WITH(NOLOCK)/*手动插入白名单*/
ON a.SESSION_ID = b.SESSION_ID
EXCEPT
SELECT JOB_NAME, STYLE, USERNAME, StarTime, SESSION_ID FROM WithoutKill_job WITH(NOLOCK);/*剔除上次留存*/
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[删除旧记录]
WITH OLD_RECORD
AS (SELECT JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID
    FROM WithoutKill_job WITH(NOLOCK)
    EXCEPT
    SELECT JOB_NAME, STYLE, USERNAME, STARTIME, a.SESSION_ID
    FROM #running AS a WITH(NOLOCK)
    INNER JOIN SessionID_WithoutKill AS b WITH(NOLOCK)
    ON a.SESSION_ID = b.SESSION_ID)
DELETE a
FROM WithoutKill_job AS a WITH(NOLOCK)
INNER JOIN OLD_RECORD AS b WITH(NOLOCK)
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.StarTime = b.StarTime
AND a.SESSION_ID = b.SESSION_ID
WHERE 1 = 1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[删除执行完毕的脚本|SessionID_WithoutKill]
DELETE FROM SessionID_WithoutKill
WHERE SESSION_ID NOT IN (SELECT SESSION_ID FROM WithoutKill_job);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[判断堵塞个数决定删除时间界限]
--①抓取超过30分钟的脚本
--②判断步骤①的个数
----大于等于10个→30分钟以内保留
----大于等于7个→45分钟以内保留
----小于7个→60分钟以内保留
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @COUNT INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SELECT @COUNT = COUNT(0)
FROM #running WITH(NOLOCK)
WHERE [运行时间] > 0.5;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[资源占用判断]
/*EXEC PROC*/
EXEC PS_BigBrother @MINUTE_FROM_NOW = 1, @QUICKSHOW = 1;/*调取此刻执行结果*/
/*TEMPDB占用最多且运行时间超过30分钟*/
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL_TEMPDB', 'U') IS NOT NULL DROP TABLE #RUNNING_KILL_TEMPDB;
WITH CTE_TEMPDB_ALLOCATIONS
AS (SELECT ROW_NUMBER()OVER(ORDER BY TEMPDB_ALLOCATIONS DESC) AS TEMPDB_ALLOCATIONS_DESC,
           START_TIME,
           LOGIN_TIME,
           RUN_DURATION,
           SESSION_ID
    FROM XTB_BIGBROTHER_LAST WITH(NOLOCK)),
     CTE_TEMPDB_CURRENT
AS (SELECT ROW_NUMBER()OVER(ORDER BY TEMPDB_CURRENT DESC) AS TEMPDB_CURRENT_DESC,
           START_TIME,
           LOGIN_TIME,
           RUN_DURATION,
           SESSION_ID
    FROM XTB_BIGBROTHER_LAST WITH(NOLOCK)),
     CTE_TEMPDB_RESULT
AS (SELECT TEMPDB_ALLOCATIONS_DESC, START_TIME, LOGIN_TIME, RUN_DURATION, SESSION_ID
    FROM CTE_TEMPDB_ALLOCATIONS WITH(NOLOCK)
    WHERE TEMPDB_ALLOCATIONS_DESC < 3
    UNION ALL
    SELECT TEMPDB_CURRENT_DESC, START_TIME, LOGIN_TIME, RUN_DURATION, SESSION_ID
    FROM CTE_TEMPDB_CURRENT WITH(NOLOCK)
    WHERE TEMPDB_CURRENT_DESC < 3)
--SELECT * FROM CTE_TEMPDB_RESULT WITH(NOLOCK)
SELECT LEFT(JOB_NAME, 4) AS JOB_NAME_BASIC,
       JOB_NAME,
       JOB_ID,
       STYLE,
       USERNAME,
       STARTIME,
       SESSION_ID,
       [等待资源类型],
       [sql执行语句],
       [运行时间],
       CURRENT_TIMESTAMP AS DEALTIME
INTO #RUNNING_KILL_TEMPDB
FROM #RUNNING WITH(NOLOCK)
WHERE SESSION_ID IN(SELECT SESSION_ID FROM CTE_TEMPDB_RESULT WITH(NOLOCK))
AND [运行时间] > 0.5;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[超时判断|TIMEIF→DELETE]
IF @COUNT >= 10
BEGIN
     DELETE FROM #running WHERE [运行时间] < 0.5;/*@COUNT >= 10→过滤小于30分钟*/
END;
ELSE IF @COUNT >= 7
BEGIN
     DELETE FROM #running WHERE [运行时间] < 0.75;/*@COUNT >= 7→过滤小于45分钟*/
END;
ELSE IF @COUNT < 7
BEGIN
     DELETE FROM #running WHERE [运行时间] < 1.00;/*@COUNT < 7→过滤小于60分钟*/
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[剔除基础作业][源头#running→生成#overtime]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CREATE|RESET]
--SELECT TOP 0 job_id INTO CHEAT_JOB_ID FROM msdb.dbo.sysjobs WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[赦免基础作业]
--判断依据:作业标识是否带数字
IF OBJECT_ID('tempdb.dbo.#overtime', 'u') IS NOT NULL DROP TABLE #overtime;
WITH JOB_NAME_CATCH
AS(SELECT LEFT(JOB_NAME, 4) AS JOB_NAME_BASIC,
          JOB_NAME,
          JOB_ID,
          STYLE,
          USERNAME,
          STARTIME,
          SESSION_ID,
          [等待资源类型],
          [sql执行语句],
          [运行时间],
          CURRENT_TIMESTAMP AS DEALTIME
FROM #running WITH(NOLOCK))
SELECT a.JOB_NAME_BASIC,
       a.JOB_NAME,
       a.JOB_ID,
       a.STYLE,
       a.USERNAME,
       a.STARTIME,
       a.SESSION_ID,
       a.[等待资源类型],
       a.[sql执行语句],
       a.[运行时间],
       a.DEALTIME
INTO #overtime
FROM JOB_NAME_CATCH AS a WITH(NOLOCK)
WHERE ISNULL(JOB_NAME_BASIC, 'NULL_INPUT') NOT LIKE '%[0-9]%';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[回插作弊作业]
IF @ANTI_THEFT = 1
BEGIN
     INSERT INTO #overtime(JOB_NAME_BASIC, JOB_NAME, JOB_ID, style, username, STARTIME, session_id, 等待资源类型, sql执行语句, 运行时间, DEALTIME)
     SELECT LEFT(a.JOB_NAME, 4) AS JOB_NAME_BASIC,
            a.JOB_NAME,
            a.JOB_ID,
            a.style,
            a.username,
            a.STARTIME,
            a.session_id,
            a.[等待资源类型],
            a.[sql执行语句],
            a.[运行时间],
            CURRENT_TIMESTAMP AS DEALTIME
     FROM #running               AS a WITH(NOLOCK)
     INNER JOIN dbo.CHEAT_JOB_ID AS b WITH(NOLOCK)
     ON a.JOB_ID = b.JOB_ID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[剔除白名单]
DELETE a
FROM #overtime             AS a WITH(NOLOCK)
INNER JOIN WithoutKill_job AS b WITH(NOLOCK)
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.STARTIME = b.STARTIME
AND a.SESSION_ID = b.SESSION_ID;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[等待资源类型|WAIT_TYPE][源头#OVERTIME→生成#RUNNING_KILL]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[必杀黑名单]
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL', 'u') IS NOT NULL DROP TABLE #RUNNING_KILL;
SELECT JOB_NAME_BASIC,
       JOB_NAME,
       JOB_ID,
       STYLE,
       USERNAME,
       STARTIME,
       SESSION_ID,
       等待资源类型,
       SQL执行语句,
       运行时间,
       DEALTIME,
       '超时终止' AS KILL_STYLE
INTO #RUNNING_KILL
FROM #OVERTIME WITH(NOLOCK)
WHERE 等待资源类型 IN ('CXPACKET', 'IO_COMPLETION', 'OLEDB', 'PAGEIOLATCH_EX', 'PAGEIOLATCH_SH', 'ASYNC_NETWORK_IO', 'ASYNC_IO_COMPLETION');
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[黑名单范围以外数量超过3个→回插名单杀死]
WITH CTE_COUNT
AS (SELECT [等待资源类型], COUNT(0) AS CNT
    FROM #OVERTIME WITH(NOLOCK)
    GROUP BY [等待资源类型]
    HAVING COUNT(0) > 3)
INSERT INTO #RUNNING_KILL(JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME, KILL_STYLE)
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID,
       a.[等待资源类型], a.[SQL执行语句], a.[运行时间], a.DEALTIME, '超时终止' AS KILL_STYLE
FROM #overtime       AS a WITH(NOLOCK)
INNER JOIN CTE_COUNT AS b WITH(NOLOCK)
ON a.[等待资源类型] = b.[等待资源类型]
EXCEPT
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME, KILL_STYLE
FROM #RUNNING_KILL;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[TEMPDB占用最多回插]
INSERT INTO #RUNNING_KILL(JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME, KILL_STYLE)
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME, '资源终止' AS KILL_STYLE
FROM #RUNNING_KILL_TEMPDB WITH(NOLOCK)
EXCEPT
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME, KILL_STYLE
FROM #RUNNING_KILL WITH(NOLOCK);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[通知优化][源头#RUNNING_KILL→生成#SEND_OA]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[生成内容]
IF OBJECT_ID('tempdb.dbo.#send_oa', 'u') IS NOT NULL DROP TABLE #send_oa;
SELECT IDENTITY(INT, 1, 1) AS ID,
       a.JOB_NAME,
       a.STYLE,
       a.USERNAME,
       a.STARTIME,
       a.SESSION_ID,
       a.[等待资源类型],
       a.USERNAME + ':通知！[SESSION_ID]号:' + CONVERT(VARCHAR(30), SESSION_ID) + '的[061]脚本超时堵塞已被KILL，麻烦优化!' + [SQL执行语句] AS NR,
       a.[运行时间],
       a.DEALTIME,
       b.USERID,
       b.PHONE
INTO #send_oa
FROM #RUNNING_KILL                 AS a WITH(NOLOCK)
INNER JOIN odsdbbi.dbo.MemberForBI AS b WITH(NOLOCK)
ON a.username = b.username;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[发送通知]
--❖❖❖❖❖❖[DECLARE]
DECLARE @CONTENT VARCHAR(MAX);
DECLARE @USERID VARCHAR(200);
DECLARE @ID INT;
--❖❖❖❖❖❖[WHILE]
WHILE EXISTS(SELECT TOP 1 * FROM #send_oa WITH(NOLOCK))
BEGIN
     SET @ID = (SELECT TOP 1 ID FROM #send_oa ORDER BY ID);
     SET @CONTENT = (SELECT NR FROM #send_oa WHERE @ID = ID);
     SET @USERID = (SELECT USERID FROM #send_oa WHERE @ID = ID);
     /*NEXT*/
     DELETE FROM #send_oa WHERE ID = @ID;
     /*SEND*/
     EXEC DATABASE_13.C6.dbo.SP_SendMessageToOAUser_Proc @CONTENT, @USERID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[操作存档记录]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DELETE|作业过程]
DELETE a
FROM dbo.KILL_RUNNINGJOB AS a
INNER JOIN #RUNNING_KILL AS b
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.SESSION_ID = b.SESSION_ID
WHERE a.HOSTNAME IS NULL/*[p_KILL_BEFORE_RESTART]结果字段HOSTNAME不为空*/
AND   b.等待资源类型 IS NOT NULL;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DELETE|临时脚本]
DELETE a
FROM dbo.KILL_RUNNINGJOB AS a
INNER JOIN #RUNNING_KILL AS b
ON ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.SESSION_ID = b.SESSION_ID
WHERE a.HOSTNAME IS NULL
AND   b.等待资源类型 IS NOT NULL
AND   b.STYLE = '临时脚本';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[INSERT]
INSERT INTO dbo.KILL_RUNNINGJOB(JOB_NAME, STYLE, USERNAME, 开始时间, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME, KILL_STYLE)
SELECT JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID,
       [等待资源类型], 
       LEFT([SQL执行语句],100) AS [SQL执行语句],
       [运行时间],
       CURRENT_TIMESTAMP AS DEALTIME,
       KILL_STYLE
FROM #RUNNING_KILL
WHERE 等待资源类型 IS NOT NULL;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[KILL操作执行]
/*DECLARE*/
DECLARE @CMD NVARCHAR(MAX);
DECLARE @SESSION_ID NVARCHAR(100);
/*WHILE*/
WHILE EXISTS (SELECT TOP 1 * FROM #RUNNING_KILL WITH(NOLOCK))
BEGIN
     SELECT TOP 1 @SESSION_ID = SESSION_ID FROM #RUNNING_KILL WITH(NOLOCK);
     SET @CMD = N'KILL ' + @SESSION_ID;
     EXEC SP_EXECUTESQL @STMT = @CMD;
     DELETE FROM #RUNNING_KILL WHERE SESSION_ID = @SESSION_ID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[删除临表]
IF OBJECT_ID('tempdb.dbo.#running', 'u') IS NOT NULL DROP TABLE #running;
IF OBJECT_ID('tempdb.dbo.#overtime', 'u') IS NOT NULL DROP TABLE #overtime;
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL', 'u') IS NOT NULL DROP TABLE #RUNNING_KILL;
--*******************************************************************************************************************************************************************
SET NOCOUNT OFF
END
```

## 2.[上传存档|20220208]-[061]-[P_RUNNING_KILL]
```SQL
USE [ODSDB]
GO
/****** Object:StoredProcedure [dbo].[P_RUNNING_KILL]    Script Date:2019-04-22 15:44:53 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROC [dbo].[P_RUNNING_KILL]
AS
BEGIN
SET NOCOUNT ON;
--*******************************************************************************************************************************************************************
--提出人员:徐海权
--创建人员:叶佩勤
--创建时间:20210830
--创建用途:监控堵塞卡顿脚本
--更新频率:每隔20分钟执行一次
--修改人员:
--报表维度:
--相关过程:P_RUNNING_RECORD(修改前20220208)
--生成结果:SELECT * FROM dbo.KILL_RUNNINGJOB WITH(NOLOCK) ORDER BY DEALTIME DESC
           /*单列不执行白名单(NOT KILL)*/--SELECT * FROM SessionID_WithoutKill WITH(NOLOCK)
--修改人员:徐海权(20220208)
--修改内容:增加防止作弊作业目录、修改判断决定删除时间界限的堵塞个数
           /*SELECT * FROM CHEAT_JOB_ID WITH(NOLOCK);*/
           /*INSERT INTO CHEAT_JOB_ID SELECT '4ED0B360-07D6-4F21-98F5-79956DDBBFE6';*/
--*******************************************************************************************************************************************************************
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[赋值变量]
/*DECLARE*/
DECLARE @WHITELIST NVARCHAR(MAX);
DECLARE @ANTI_THEFT INT;
/*SET*/
SET @WHITELIST = N'Administrator|BIGDATA|104282';
SET @ANTI_THEFT = 1;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[获取最新执行情况]
EXEC P_ALLRUN /*@BREAKPOINT_INPUT*/0, /*@SHOW_OR_NOT_INPUT*/0;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[基本信息整理][生成#RUNNING]
IF OBJECT_ID('tempdb.dbo.#running', 'u') IS NOT NULL DROP TABLE #running;
SELECT ROW_NUMBER()OVER(ORDER BY a.开始时间) AS NUM,
       b.JOB_ID,
       b.Job_Name,
       a.loginame,
       CASE WHEN LEN(b.Job_Name) > 0 THEN '执行作业' ELSE '临时脚本' END AS style,
       CASE WHEN a.username IS NULL THEN c.JOB_OWNER ELSE a.username END AS username,
       a.开始时间 AS STARTIME,
       a.session_id,
       a.等待资源类型,
       a.[sql执行语句],
       DATEDIFF(MINUTE, a.开始时间, GETDATE()) * 1.00 / 60 AS 运行时间
INTO #running
FROM dbo.XTB_ALL_RUN                         AS a WITH(NOLOCK)
LEFT OUTER JOIN dbo.sysjob_running_sessionid AS b WITH(NOLOCK)
ON a.SESSION_ID = b.SESSIONID
LEFT OUTER JOIN dbo.XTB_sysjobs_owner_output AS c WITH(NOLOCK)
ON b.JOB_ID = c.JOB_ID;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[名单处理]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[放行白名单(大数据|作业执行|)]
WITH CTE_STRING_SPLIT
AS (SELECT VALUE
    FROM STRING_SPLIT(@WHITELIST, '|'))
    --FROM STRING_SPLIT('Administrator|BIGDATA|104282', '|'))
DELETE a
FROM #running               AS a WITH(NOLOCK)
CROSS JOIN CTE_STRING_SPLIT AS b WITH(NOLOCK)
WHERE CHARINDEX(VALUE, loginame, 1) > 0;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[新增入仓白名单|TABLE|SessionID_WithoutKill]
/*单列不执行白名单(NOT KILL)*/--SELECT * FROM SessionID_WithoutKill WITH(NOLOCK)
INSERT INTO WithoutKill_job(JOB_NAME, STYLE, USERNAME, StarTime, SESSION_ID)
SELECT a.job_name, a.style, a.username, a.StarTime, a.session_id
FROM #running                    AS a WITH(NOLOCK)
INNER JOIN SessionID_WithoutKill AS b WITH(NOLOCK)/*手动插入白名单*/
ON a.SESSION_ID = b.SESSION_ID
EXCEPT
SELECT JOB_NAME, STYLE, USERNAME, StarTime, SESSION_ID FROM WithoutKill_job WITH(NOLOCK);/*剔除上次留存*/
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[删除旧记录]
WITH OLD_RECORD
AS (SELECT JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID
    FROM WithoutKill_job WITH(NOLOCK)
    EXCEPT
    SELECT JOB_NAME, STYLE, USERNAME, STARTIME, a.SESSION_ID
    FROM #running AS a WITH(NOLOCK)
    INNER JOIN SessionID_WithoutKill AS b WITH(NOLOCK)
    ON a.SESSION_ID = b.SESSION_ID)
DELETE a
FROM WithoutKill_job AS a WITH(NOLOCK)
INNER JOIN OLD_RECORD AS b WITH(NOLOCK)
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.StarTime = b.StarTime
AND a.SESSION_ID = b.SESSION_ID
WHERE 1 = 1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[删除执行完毕的脚本|SessionID_WithoutKill]
DELETE FROM SessionID_WithoutKill
WHERE SESSION_ID NOT IN (SELECT SESSION_ID FROM WithoutKill_job);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[判断堵塞个数决定删除时间界限]
--①抓取超过30分钟的脚本
--②判断步骤①的个数
----大于等于10个→30分钟以内保留
----大于等于7个→45分钟以内保留
----小于7个→60分钟以内保留
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @COUNT INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SELECT @COUNT = COUNT(0)
FROM #running WITH(NOLOCK)
WHERE [运行时间] > 0.5;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[IF→DELETE]
IF @COUNT >= 10
BEGIN
     DELETE FROM #running WHERE [运行时间] < 0.5;/*@COUNT >= 10→过滤小于30分钟*/
END;
ELSE IF @COUNT >= 7
BEGIN
     DELETE FROM #running WHERE [运行时间] < 0.75;/*@COUNT >= 7→过滤小于45分钟*/
END;
ELSE IF @COUNT < 7
BEGIN
     DELETE FROM #running WHERE [运行时间] < 1.00;/*@COUNT < 7→过滤小于60分钟*/
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[剔除基础作业][源头#running→生成#overtime]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CREATE|RESET]
--SELECT TOP 0 job_id INTO CHEAT_JOB_ID FROM msdb.dbo.sysjobs WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[赦免基础作业]
--判断依据:作业标识是否带数字
IF OBJECT_ID('tempdb.dbo.#overtime', 'u') IS NOT NULL DROP TABLE #overtime;
WITH JOB_NAME_CATCH
AS(SELECT LEFT(JOB_NAME, 4) AS JOB_NAME_BASIC,
          JOB_NAME,
          JOB_ID,
          STYLE,
          USERNAME,
          STARTIME,
          SESSION_ID,
          [等待资源类型],
          [sql执行语句],
          [运行时间],
          CURRENT_TIMESTAMP AS DEALTIME
FROM #running WITH(NOLOCK))
SELECT a.JOB_NAME_BASIC,
       a.JOB_NAME,
       a.JOB_ID,
       a.STYLE,
       a.USERNAME,
       a.STARTIME,
       a.SESSION_ID,
       a.[等待资源类型],
       a.[sql执行语句],
       a.[运行时间],
       a.DEALTIME
INTO #overtime
FROM JOB_NAME_CATCH AS a WITH(NOLOCK)
WHERE ISNULL(JOB_NAME_BASIC, 'NULL_INPUT') NOT LIKE '%[0-9]%';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[回插作弊作业]
IF @ANTI_THEFT = 1
BEGIN
     INSERT INTO #overtime(JOB_NAME_BASIC, JOB_NAME, JOB_ID, style, username, STARTIME, session_id, 等待资源类型, sql执行语句, 运行时间, DEALTIME)
     SELECT LEFT(a.JOB_NAME, 4) AS JOB_NAME_BASIC,
            a.JOB_NAME,
            a.JOB_ID,
            a.style,
            a.username,
            a.STARTIME,
            a.session_id,
            a.[等待资源类型],
            a.[sql执行语句],
            a.[运行时间],
            CURRENT_TIMESTAMP AS DEALTIME
     FROM #running               AS a WITH(NOLOCK)
     INNER JOIN dbo.CHEAT_JOB_ID AS b WITH(NOLOCK)
     ON a.JOB_ID = b.JOB_ID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[剔除白名单]
DELETE a
FROM #overtime             AS a WITH(NOLOCK)
INNER JOIN WithoutKill_job AS b WITH(NOLOCK)
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.STARTIME = b.STARTIME
AND a.SESSION_ID = b.SESSION_ID;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[等待资源类型|WAIT_TYPE][源头#OVERTIME→生成#RUNNING_KILL]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[必杀黑名单]
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL', 'u') IS NOT NULL DROP TABLE #RUNNING_KILL;
SELECT JOB_NAME_BASIC,
       JOB_NAME,
       JOB_ID,
       STYLE,
       USERNAME,
       STARTIME,
       SESSION_ID,
       等待资源类型,
       SQL执行语句,
       运行时间,
       DEALTIME
INTO #RUNNING_KILL
FROM #OVERTIME WITH(NOLOCK)
WHERE 等待资源类型 IN ('CXPACKET', 'IO_COMPLETION', 'OLEDB', 'PAGEIOLATCH_EX', 'PAGEIOLATCH_SH', 'ASYNC_NETWORK_IO', 'ASYNC_IO_COMPLETION');
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[黑名单范围以外数量超过3个→回插名单杀死]
WITH CTE_COUNT
AS (SELECT [等待资源类型], COUNT(0) AS CNT
    FROM #OVERTIME WITH(NOLOCK)
    GROUP BY [等待资源类型]
    HAVING COUNT(0) > 3)
INSERT INTO #RUNNING_KILL(JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME)
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID,
       a.[等待资源类型], a.[SQL执行语句], a.[运行时间], a.DEALTIME
FROM #overtime       AS a WITH(NOLOCK)
INNER JOIN CTE_COUNT AS b WITH(NOLOCK)
ON a.[等待资源类型] = b.[等待资源类型]
EXCEPT
SELECT JOB_NAME_BASIC, JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID, [等待资源类型], [sql执行语句], [运行时间], DEALTIME
FROM #RUNNING_KILL;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[通知优化][源头#RUNNING_KILL→生成#SEND_OA]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[生成内容]
IF OBJECT_ID('tempdb.dbo.#send_oa', 'u') IS NOT NULL DROP TABLE #send_oa;
SELECT IDENTITY(INT, 1, 1) AS ID,
       a.JOB_NAME,
       a.STYLE,
       a.USERNAME,
       a.STARTIME,
       a.SESSION_ID,
       a.[等待资源类型],
       a.USERNAME + ':通知！[SESSION_ID]号:' + CONVERT(VARCHAR(30), SESSION_ID) + '的[061]脚本超时堵塞已被KILL，麻烦优化!' + [SQL执行语句] AS NR,
       a.[运行时间],
       a.DEALTIME,
       b.USERID,
       b.PHONE
INTO #send_oa
FROM #RUNNING_KILL                 AS a WITH(NOLOCK)
INNER JOIN odsdbbi.dbo.MemberForBI AS b WITH(NOLOCK)
ON a.username = b.username;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[发送通知]
--❖❖❖❖❖❖[DECLARE]
DECLARE @CONTENT VARCHAR(MAX);
DECLARE @USERID VARCHAR(200);
DECLARE @ID INT;
--❖❖❖❖❖❖[WHILE]
WHILE EXISTS(SELECT TOP 1 * FROM #send_oa WITH(NOLOCK))
BEGIN
     SET @ID = (SELECT TOP 1 ID FROM #send_oa ORDER BY ID);
     SET @CONTENT = (SELECT NR FROM #send_oa WHERE @ID = ID);
     SET @USERID = (SELECT USERID FROM #send_oa WHERE @ID = ID);
     /*NEXT*/
     DELETE FROM #send_oa WHERE ID = @ID;
     /*SEND*/
     EXEC DATABASE_13.C6.dbo.SP_SendMessageToOAUser_Proc @CONTENT, @USERID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[操作存档记录]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DELETE|作业过程]
DELETE a
FROM dbo.KILL_RUNNINGJOB AS a
INNER JOIN #RUNNING_KILL AS b
ON  ISNULL(a.JOB_NAME, 0) = ISNULL(b.JOB_NAME, 0)
AND ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.SESSION_ID = b.SESSION_ID
WHERE a.HOSTNAME IS NULL/*[p_KILL_BEFORE_RESTART]结果字段HOSTNAME不为空*/
AND   b.等待资源类型 IS NOT NULL;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DELETE|临时脚本]
DELETE a
FROM dbo.KILL_RUNNINGJOB AS a
INNER JOIN #RUNNING_KILL AS b
ON ISNULL(a.USERNAME, 0) = ISNULL(b.USERNAME, 0)
AND a.SESSION_ID = b.SESSION_ID
WHERE a.HOSTNAME IS NULL
AND   b.等待资源类型 IS NOT NULL
AND   b.STYLE = '临时脚本';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[INSERT]
INSERT INTO dbo.KILL_RUNNINGJOB(JOB_NAME, STYLE, USERNAME, 开始时间, SESSION_ID, [等待资源类型], [SQL执行语句], [运行时间], DEALTIME)
SELECT JOB_NAME, STYLE, USERNAME, STARTIME, SESSION_ID,
       [等待资源类型], 
       LEFT([SQL执行语句],100) AS [SQL执行语句],
       [运行时间],
       CURRENT_TIMESTAMP AS DEALTIME
FROM #RUNNING_KILL
WHERE 等待资源类型 IS NOT NULL;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[KILL操作执行]
/*DECLARE*/
DECLARE @CMD NVARCHAR(MAX);
DECLARE @SESSION_ID NVARCHAR(100);
/*WHILE*/
WHILE EXISTS (SELECT TOP 1 * FROM #RUNNING_KILL WITH(NOLOCK))
BEGIN
     SELECT TOP 1 @SESSION_ID = SESSION_ID FROM #RUNNING_KILL WITH(NOLOCK);
     SET @CMD = N'KILL ' + @SESSION_ID;
     EXEC SP_EXECUTESQL @STMT = @CMD;
     DELETE FROM #RUNNING_KILL WHERE SESSION_ID = @SESSION_ID;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[删除临表]
IF OBJECT_ID('tempdb.dbo.#running', 'u') IS NOT NULL DROP TABLE #running;
IF OBJECT_ID('tempdb.dbo.#overtime', 'u') IS NOT NULL DROP TABLE #overtime;
IF OBJECT_ID('tempdb.dbo.#RUNNING_KILL', 'u') IS NOT NULL DROP TABLE #RUNNING_KILL;
--*******************************************************************************************************************************************************************
SET NOCOUNT OFF
END
```

## 3.[上传存档|20220323]-[000]-[GETJOBIDFROMPROGRAMNAME]
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[测试结果]
SELECT *
FROM msdb.dbo.sysjobs WITH(NOLOCK)
WHERE job_id = dbo.GetJobIdFromProgramName('SQLAgent - TSQL JobStep (Job 0xECC4D10D8CFD94458B0B7640AB35D37A : Step 9)');
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
```