---
{"dg-publish":true,"permalink":"/2-021/20220413-061-der-store-xs-switch-proc/"}
---


# <font color=#DC143C>(20220413)-(过程解析)-(061)-(基础)-(DER_STORE_XS_SWITCH_PROC)</font>
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

```toc
```

## 逻辑压缩
1. 搭建总表
   1. 数据来源
      1. 分区店表|`odsdbfq.dbo.store AS a WITH(NOLOCK)`
      2. 广东店表|`odsdb.dbo.store AS b WITH(NOLOCK)`
   2. 关联条件:`ON a.OLDCODE = b.CODE;`
   3. 处理方法:`CASE WHEN a.isnewcode IS NULL THEN '升级店' ELSE '新开店' AS END memo`
      1. `isnewcode`空值→升级店
      2. `isnewcode`非空→新开店
2. 取值销售额
   1. 数据来源:
      1. 广东销售|`FROM odsdb.dbo.m_stsale a WITH(NOLOCK)`
      2. 分区销售|`FROM odsdbfq.dbo.m_stsale AS a WITH(NOLOCK)`
   2. 过滤条件:
      1. 关联处理步骤[1]结果`#main`→选取交集
      2. `AND memo = '有效'`→有效
      3. `AND CHARINDEX('异常', ISNULL(memo2, 0)) = 0`→剔除异常
      4. `ABS(a.LKS - b.LKS_AVG) <= b.LKS_STDEVP * 2`→<strong><font color=#FF0000>来客数选择2个标准差以内</font></strong>
   3. 处理结果
      1. 更新字段`gm_sale_max_day`
      2. 更新字段`xs_sale_min_day`
      3. 更新字段`switch_day`
         1. 处理方法:`SET switch_day = (CASE WHEN xs_sale_min_day <> xs_pradate THEN xs_sale_min_day ELSE xs_pradate END);`
         2. 逻辑总结:<strong><font color=#FF4500>鲜食销售最早日期不等于开业日期，优先取值鲜食销售最早日期</font></strong>
3. 经营时段
   1. 广美最大销售日期小于等于鲜食最小销售日期→正常切换
      1. 处理结果
         1. `广美最早日期-广美最晚日期`
         2. `鲜食最早日期-鲜食最晚日期`
      2. 广美最大销售日期大于鲜食最小销售日期
         1. `广美最早日期-鲜食最早日期`
         2. `鲜食最早日期-鲜食最晚日期`
         3. `鲜食最晚日期-广美最晚日期`→来回切换
4. 处理步骤[3]：营业时间拼接
5. 判断切换类型
   1. `鲜食最早日期 = 广美最晚日期`→无缝切换
   2. `鲜食最早日期 > 广美最晚日期`→间隔切换
   3. `鲜食最早日期 < 广美最晚日期`且`存在时段'鲜食最晚日期-广美最晚日期'`→来回切换
   4. `鲜食不存在销售流水`→尚未切换
   5. `广美旧店号不存在销售流水`→借号切换
   6. 其他→销售异常
6. 灾难备份更新
    1. 备份
    2. 清空
    3. 全量更新
    4. 更新失败→恢复
7. 发送监控
    1. 调取积木:[[(2_021)(原子熔炼)/√(20220326)-(积木问题)-(000)-(NP)-(STRING_SPLIT多列分割)|STRING_SPLIT多列分割]]

## (DER_STORE_XS_SWITCH_PROC)-(20220413)-(061)-(鲜食)
```SQL
USE [odsdbbi]
GO
/****** Object:StoredProcedure [dbo].[DER_STORE_XS_SWITCH_PROC]    Script Date:2019-04-22 15:44:53 ******//*[2]❆❆❆❆❆*/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROC [dbo].[DER_STORE_XS_SWITCH_PROC](@MESSAGE_INPUT NVARCHAR(1000))
AS
BEGIN
SET NOCOUNT ON
--*******************************************************************************************************************************************************************
--属服务器:[192.168.0.61]
--属数据库:[odsdbbi]
--提出人员:徐海权
--创建人员:罗清伟
--创建时间:20220413
--创建用途:鲜食切换店日期判定
--更新频率:每日
--修改人员:徐海权
--修改内容:剔除异常销售日期|变更销售日期展示形式
--报表维度:
--相关过程:[旧过程|store_xs_switch_proc]
--生成结果:SELECT * FROM odsdbbi.dbo.store_xs_switch WITH(NOLOCK)ORDER BY XS_CODE;
           --SELECT * FROM odsdbbi.dbo.store_xs_switch WITH(NOLOCK)ORDER BY switch_style;
--*******************************************************************************************************************************************************************
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[#main]
IF OBJECT_ID('tempdb.dbo.#main', 'U') IS NOT NULL DROP TABLE #main;
SELECT b.CODE AS GM_CODE,
       b.GID AS GM_GID,
       a.CODE AS XS_CODE,
       a.GID AS XS_GID,
       CONVERT(DATETIME, NULL) AS switch_day,
       a.pradate AS xs_pradate,
       CONVERT(DATETIME, NULL) AS xs_sale_min_day,
       CONVERT(DATETIME, NULL) AS gm_sale_max_day,
        a.stat AS xs_stat,
        b.stat AS gm_stat,
        CONVERT(VARCHAR(20), NULL) AS switch_style,
        CONVERT(VARCHAR(200), NULL) AS time_list,
        CASE WHEN a.isnewcode IS NULL THEN '升级店' ELSE '新开店' END AS memo
INTO #main
FROM odsdbfq.dbo.store AS a WITH(NOLOCK)
INNER JOIN odsdb.dbo.store AS b WITH(NOLOCK)
ON a.OLDCODE = b.CODE;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[INDEX]
CREATE NONCLUSTERED INDEX idx_GM_CODE ON #main(GM_CODE);
CREATE NONCLUSTERED INDEX IDX_XS_CODE ON #main(XS_CODE);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[销售判定]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[GD|#sale_day]
IF OBJECT_ID('tempdb.dbo.#sale_day', 'U') IS NOT NULL DROP TABLE #sale_day;
WITH GD_M_STSALE
AS(SELECT RQ, CODE, LKS, XSE, memo, memo2
   FROM odsdb.dbo.m_stsale AS a WITH(NOLOCK)
   WHERE EXISTS(SELECT *
                FROM #main AS b WITH(NOLOCK)
                WHERE a.CODE = b.GM_CODE)),
CTE_AGG
AS(SELECT CODE, STDEVP(LKS) AS LKS_STDEVP, AVG(LKS) AS LKS_AVG
   FROM GD_M_STSALE WITH(NOLOCK)
   GROUP BY CODE)
SELECT a.code,
       MAX(a.rq) AS max_rq,
       MIN(a.rq) AS min_rq
INTO #sale_day
FROM GD_M_STSALE AS a WITH(NOLOCK)
INNER JOIN CTE_AGG AS b WITH(NOLOCK)
ON a.CODE = b.CODE
WHERE ABS(a.LKS - b.LKS_AVG) <= b.LKS_STDEVP * 2
AND a.memo = '有效'
AND CHARINDEX('异常', ISNULL(a.memo2, 0)) = 0
GROUP BY a.code;
--∷∷∷∷∷∷[UPDATE]
UPDATE a
SET a.gm_sale_max_day = b.max_rq
FROM #main AS a WITH(NOLOCK), #sale_day AS b WITH(NOLOCK)
WHERE a.GM_CODE = b.CODE;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[FQ|#sale_xs_day]
IF OBJECT_ID('tempdb.dbo.#sale_xs_day', 'U') IS NOT NULL DROP TABLE #sale_xs_day;
WITH XS_M_STSALE
AS(SELECT RQ, CODE, LKS, XSE, memo--, memo2
   FROM odsdbfq.dbo.m_stsale AS a WITH(NOLOCK)
   WHERE EXISTS(SELECT *
             FROM #main AS b WITH(NOLOCK)
             WHERE a.CODE = b.XS_CODE)),
CTE_AGG
AS(SELECT CODE, STDEVP(LKS) AS LKS_STDEVP, AVG(LKS) AS LKS_AVG
   FROM XS_M_STSALE WITH(NOLOCK)
   GROUP BY CODE)
SELECT a.code,
       MAX(a.rq) AS max_rq,
       MIN(a.rq) AS min_rq
INTO #sale_xs_day
FROM XS_M_STSALE AS a WITH(NOLOCK)
INNER JOIN CTE_AGG AS b WITH(NOLOCK)
ON a.CODE = b.CODE
WHERE ABS(a.LKS - b.LKS_AVG) <= b.LKS_STDEVP * 2
AND a.memo = '有效'
--AND CHARINDEX('异常', ISNULL(a.memo2, 0)) = 0
GROUP BY a.code;
--SELECT code,
--       MAX(rq) max_rq,
--       MIN(rq) min_rq
--INTO #sale_xs_day
--FROM odsdbfq.dbo.m_stsale AS a WITH(NOLOCK)
--WHERE EXISTS (SELECT *
--              FROM #main AS b WITH(NOLOCK)
--              WHERE a.code = b.XS_CODE)
--AND memo = '有效'
----AND CHARINDEX('异常', ISNULL(memo2, 0)) = 0/*分区缺失*/
--GROUP BY code;
--∷∷∷∷∷∷[UPDATE]
UPDATE a
SET a.xs_sale_min_day = b.min_rq
FROM #main a, #sale_xs_day b
WHERE a.XS_CODE = b.CODE;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[UPDATE|switch_day]
UPDATE #main
SET switch_day = (CASE WHEN xs_sale_min_day <> xs_pradate THEN xs_sale_min_day ELSE xs_pradate END);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[#TIME]
IF OBJECT_ID('tempdb.dbo.#TIME', 'U') IS NOT NULL DROP TABLE #TIME;
CREATE TABLE #TIME (CODE VARCHAR(100),
                    TIME_LIST VARCHAR(100));
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[1|广美最大销售日期小于等于鲜食最小销售日期]
--∷∷∷∷∷∷[广美最早日期-广美最晚日期]
INSERT INTO #TIME
SELECT a.GM_CODE,
       'GM-GM/' + CONVERT(VARCHAR(50), b.min_rq, 112) + '-' + CONVERT(VARCHAR(50), b.max_rq, 112)
FROM #main AS a WITH(NOLOCK)
INNER JOIN #sale_day AS b WITH(NOLOCK)
ON a.GM_CODE = b.CODE AND a.xs_sale_min_day >= b.max_rq;
--∷∷∷∷∷∷[鲜食最早日期-鲜食最晚日期]
INSERT INTO #TIME
SELECT a.GM_CODE,
       'XS-XS/' + CONVERT(VARCHAR(50), a.xs_sale_min_day, 112) + '-' + CONVERT(VARCHAR(50), c.max_rq, 112)
FROM #main AS a WITH(NOLOCK)
INNER JOIN #sale_day AS b WITH(NOLOCK)
ON a.GM_CODE = b.CODE AND a.xs_sale_min_day >= b.max_rq
LEFT OUTER JOIN #sale_xs_day AS c WITH(NOLOCK)
ON a.XS_CODE = c.CODE;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[2|广美最大销售日期大于鲜食最小销售日期]
--∷∷∷∷∷∷[广美最早日期-鲜食最早日期]
INSERT INTO #TIME
SELECT a.GM_CODE,
       'GM-XS/' + CONVERT(VARCHAR(50), b.min_rq, 112) + '-' + CONVERT(VARCHAR(50), c.min_rq, 112)
FROM #main AS a WITH(NOLOCK)
INNER JOIN #sale_day AS b WITH(NOLOCK)
ON a.GM_CODE = b.CODE AND a.xs_sale_min_day < b.max_rq
LEFT OUTER JOIN #sale_xs_day AS c WITH(NOLOCK)
ON a.XS_CODE = c.CODE;
--∷∷∷∷∷∷[鲜食最早日期-鲜食最晚日期]
INSERT INTO #TIME
SELECT a.GM_CODE,
       'XS-XS/' + CONVERT(VARCHAR(50), c.min_rq, 112) + '-' + CONVERT(VARCHAR(50), c.max_rq, 112)
FROM #main AS a WITH(NOLOCK)
INNER JOIN #sale_day AS b WITH(NOLOCK)
ON a.GM_CODE = b.CODE AND a.xs_sale_min_day < max_rq
LEFT OUTER JOIN #sale_xs_day AS c WITH(NOLOCK)
ON a.XS_CODE = c.CODE;
--∷∷∷∷∷∷[鲜食最晚日期-广美最晚日期]
INSERT INTO #TIME
SELECT a.GM_CODE,
       'XS-GM/' + CONVERT(VARCHAR(50), c.max_rq, 112) + '-' + CONVERT(VARCHAR(50), b.max_rq, 112)
FROM #main AS a WITH(NOLOCK)
INNER JOIN #sale_day AS b WITH(NOLOCK)
ON a.GM_CODE = b.CODE AND a.xs_sale_min_day < max_rq
LEFT OUTER JOIN #sale_xs_day AS c WITH(NOLOCK)
ON a.XS_CODE = c.CODE
WHERE b.max_rq > c.max_rq;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[营业时间拼接]
WITH FOR_XML_PATH AS
(SELECT b.CODE, STUFF((SELECT '|' + TIME_LIST
                       FROM #TIME a WITH(NOLOCK)
                       WHERE a.CODE = b.CODE
                       FOR XML PATH('')), 1, 1, '') AS TIME_LIST
 FROM #TIME AS b WITH(NOLOCK)
 GROUP BY b.CODE)
UPDATE a
SET a.TIME_LIST = b.TIME_LIST
FROM #main AS a WITH(NOLOCK)
INNER JOIN FOR_XML_PATH AS b WITH(NOLOCK)
ON a.GM_CODE = b.CODE;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[判断切换类型]
UPDATE #main
SET switch_style = (CASE WHEN xs_sale_min_day = gm_sale_max_day--鲜食最早日期 = 广美最晚日期
                         THEN '无缝切换'
                         WHEN xs_sale_min_day > gm_sale_max_day--鲜食最早日期 > 广美最晚日期
                         THEN '间隔切换'
                         WHEN xs_sale_min_day < gm_sale_max_day AND CHARINDEX('XS-GM', TIME_LIST, 1) > 0--鲜食最早日期 < 广美最晚日期 & 存在时段'鲜食最晚日期-广美最晚日期'
                         THEN '来回切换'
                         WHEN xs_sale_min_day IS NULL
                         THEN '尚未切换'
                         WHEN gm_sale_max_day IS NULL
                         THEN '借号切换'
                         ELSE '销售异常'
                    END);
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[灾难备份]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[备份]
IF OBJECT_ID('odsdbbi.dbo.store_xs_switch_bak', 'U') IS NOT NULL DROP TABLE odsdbbi.dbo.store_xs_switch_bak;
SELECT GM_CODE, GM_GID,
       XS_CODE, XS_GID,
       SWITCH_DAY,
       XS_PRADATE,
       XS_SALE_MIN_DAY, GM_SALE_MAX_DAY,
       XS_STAT, GM_STAT,
       SWITCH_STYLE, TIME_LIST, UPDATETIME, MEMO
INTO odsdbbi.dbo.store_xs_switch_bak
FROM odsdbbi.dbo.store_xs_switch WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[清空]
TRUNCATE TABLE odsdbbi.dbo.store_xs_switch;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[更新]
INSERT INTO odsdbbi.dbo.store_xs_switch(GM_CODE, GM_GID, XS_CODE, XS_GID, switch_day, xs_pradate, xs_sale_min_day,
                                        gm_sale_max_day, xs_stat, gm_stat, switch_style, time_list, updatetime, memo)
SELECT GM_CODE, GM_GID, XS_CODE, XS_GID, switch_day, xs_pradate, xs_sale_min_day, gm_sale_max_day, xs_stat, gm_stat,
       switch_style, time_list, GETDATE() AS updatetime, memo
FROM #main WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[恢复]
IF(SELECT COUNT(0)FROM odsdbbi.dbo.store_xs_switch WITH(NOLOCK)) = 0
BEGIN
     INSERT INTO odsdbbi.dbo.store_xs_switch(GM_CODE, GM_GID, XS_CODE, XS_GID, switch_day, xs_pradate, xs_sale_min_day,
                                             gm_sale_max_day, xs_stat, gm_stat, switch_style, time_list, updatetime, memo)
     SELECT GM_CODE, GM_GID, XS_CODE, XS_GID, switch_day, xs_pradate, xs_sale_min_day, gm_sale_max_day, xs_stat, gm_stat,
            switch_style, time_list, updatetime, memo
     FROM odsdbbi.dbo.store_xs_switch_bak WITH(NOLOCK);
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[发送监控]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @INPUT NVARCHAR(MAX);
DECLARE @NUM_CNT INT;
DECLARE @NUM_MAX INT;
DECLARE @VALUE_LEFT NVARCHAR(MAX);
DECLARE @VALUE_RIGHT NVARCHAR(MAX);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @INPUT = N'3457-苏润铭|102471-张美芳|1076-席雪影|300169-陈志伟';
SET @INPUT = @MESSAGE_INPUT;
SET @NUM_CNT = 1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SELECT→#TABLE_1]
IF OBJECT_ID('tempdb.dbo.#TABLE_1', 'U') IS NOT NULL DROP TABLE #TABLE_1;
WITH STEP_1
AS (SELECT VALUE,
    LEN(VALUE) AS VALUE_LENGTH,
    CHARINDEX('-', VALUE, 1) AS STEP_2_BASE
    FROM STRING_SPLIT(@INPUT, '|'))
SELECT ROW_NUMBER()OVER(ORDER BY(SELECT NULL)) AS NUM,
       VALUE,
       LEFT(VALUE, STEP_2_BASE - 1) AS VALUE_LEFT,
       SUBSTRING(VALUE, STEP_2_BASE + 1, VALUE_LENGTH - STEP_2_BASE) AS VALUE_RIGHT
INTO #TABLE_1
FROM STEP_1 WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[#TIP]
IF OBJECT_ID('tempdb.dbo.#TIP', 'U') IS NOT NULL DROP TABLE #TIP;
SELECT GM_CODE,
       USERID,
       PHONE,
       '温馨提示：' + '广美店号：' + RTRIM(GM_CODE) + ',鲜食店号：' + RTRIM(XS_CODE) + ',存在双方都为有效店，请及时修正谢谢！' AS NEIRONG
INTO #TIP
FROM odsdbbi.dbo.store_xs_switch AS a WITH(NOLOCK)
CROSS JOIN odsdb.dbo.user_dept_detail AS b WITH(NOLOCK)
--ON 1 = 1
INNER JOIN #TABLE_1 AS c WITH(NOLOCK)
ON b.USERID = c.VALUE_LEFT
WHERE gm_stat = '有效'
AND   xs_stat = '有效'
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SEND]
WHILE EXISTS(SELECT TOP 1 * FROM #TIP WITH(NOLOCK))
BEGIN
     --∷∷∷∷∷∷[DECLARE]
     DECLARE @GM_CODE VARCHAR(200);
     DECLARE @USERID VARCHAR(200);
     DECLARE @NEIRONG VARCHAR(MAX);
     --∷∷∷∷∷∷[SET]
     SET @GM_CODE = (SELECT TOP 1 GM_CODE FROM #TIP WITH(NOLOCK));
     SET @USERID = (SELECT TOP 1 USERID FROM #TIP WHERE GM_CODE = @GM_CODE);
     SET @NEIRONG = (SELECT TOP 1 NEIRONG FROM #TIP WHERE GM_CODE = @GM_CODE AND USERID = @USERID);
     --∷∷∷∷∷∷[SEND]
     EXEC DATABASE_13.C6.DBO.SP_SENDMESSAGETOOAUSER_PROC @NEIRONG, @USERID;
     --∷∷∷∷∷∷[DELETE]
     DELETE FROM #TIP WHERE GM_CODE = @GM_CODE AND USERID = @USERID;
END
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[注释]
--EXEC sp_addextendedproperty 'MS_Description', '鲜食店切换日期表', 'user', dbo, 'table', 'store_xs_switch';
--EXEC sp_addextendedproperty 'MS_Description', '广美gid', 'user', dbo, 'table', 'store_xs_switch', 'column', GM_gid;
--EXEC sp_addextendedproperty 'MS_Description', '鲜食code', 'user', dbo, 'table', 'store_xs_switch', 'column', XS_CODE;
--EXEC sp_addextendedproperty 'MS_Description', '鲜食gid', 'user', dbo, 'table', 'store_xs_switch', 'column', XS_GID;
--EXEC sp_addextendedproperty 'MS_Description', '切换日期', 'user', dbo, 'table', 'store_xs_switch', 'column', switch_day;
--EXEC sp_addextendedproperty 'MS_Description', '鲜食开业日期', 'user', dbo, 'table', 'store_xs_switch', 'column', xs_pradate;
--EXEC sp_addextendedproperty 'MS_Description', '鲜食销售最小日期', 'user', dbo, 'table', 'store_xs_switch', 'column', xs_sale_min_day;
--EXEC sp_addextendedproperty 'MS_Description', '广美销售最大日期', 'user', dbo, 'table', 'store_xs_switch', 'column', gm_sale_max_day;
--EXEC sp_addextendedproperty 'MS_Description', '鲜食有效状态', 'user', dbo, 'table', 'store_xs_switch', 'column', xs_stat;
--EXEC sp_addextendedproperty 'MS_Description', '广美有效状态', 'user', dbo, 'table', 'store_xs_switch', 'column', gm_stat;
--EXEC sp_addextendedproperty 'MS_Description', '切换类型', 'user', dbo, 'table', 'store_xs_switch', 'column', switch_style;
--EXEC sp_addextendedproperty 'MS_Description', '营运时间（左为广美，右为鲜食）', 'user', dbo, 'table', 'store_xs_switch', 'column', time_list;
--*******************************************************************************************************************************************************************
SET NOCOUNT OFF
END
```