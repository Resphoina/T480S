---
{"dg-publish":true,"permalink":"/(2_021)(单表标注)/√(20211211)-(单表标注)-(000)-(数仓)-(XTB_PARM_WAREHOUSE-参数调用中枢表)/"}
---


# <font color=#DC143C>(20211211)-(单表标注)-(000)-(数仓)-(XTB_PARM_WAREHOUSE-参数调用中枢表)</font>
URL:: 

| 萃取函数 | 萃取解法 |
| ---- | ---- |
| \-   | \-   |


```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[参数调用中枢表|dbo.XTB_PARM_WAREHOUSE]
CREATE TABLE XTB_PARM_WAREHOUSE(UUID UNIQUEIDENTIFIER DEFAULT NEWID(),
                                PROCNAME VARCHAR(1000),/*过程名称*/
                                PARMNAME VARCHAR(1000),/*参数名称*/
                                PARMVALUE VARCHAR(1000),/*参数数值*/
                                ADDTIME DATETIME DEFAULT CURRENT_TIMESTAMP,
                                ADMINID sysname DEFAULT SUSER_SNAME());
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[维护案例]
INSERT INTO dbo.XTB_PARM_WAREHOUSE(PROCNAME, PARMNAME, PARMVALUE)
SELECT 'p_staff_nameforshort', '@Build_On_People_INPUT', '邓珊|徐海权|赵必胜|陈浩平|周焕焕';
GO
INSERT INTO dbo.XTB_PARM_WAREHOUSE(PROCNAME, PARMNAME, PARMVALUE)
SELECT 'p_staff_nameforshort', '@MONITOR_SEND_TO_INPUT', '102470';
GO
INSERT INTO dbo.XTB_PARM_WAREHOUSE(PROCNAME, PARMNAME, PARMVALUE)
SELECT 'p_staff_nameforshort', '@BREAKPOINT_INPUT', '0';
GO
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[调用案例]
WITH ADDTIME_MAX
AS (SELECT PROCNAME, PARMNAME, MAX(ADDTIME) AS ADDTIME_MAX
    FROM dbo.XTB_PARM_WAREHOUSE WITH(NOLOCK)
    WHERE PROCNAME = 'P_STAFF_NAMEFORSHORT'
      AND PARMNAME = '@Build_On_People_INPUT'
    GROUP BY PROCNAME, PARMNAME)
SELECT PARMVALUE
FROM dbo.XTB_PARM_WAREHOUSE AS a WITH(NOLOCK)
INNER JOIN ADDTIME_MAX AS b WITH(NOLOCK)
ON a.PROCNAME = b.PROCNAME
AND a.PARMNAME = b.PARMNAME
AND a.ADDTIME = b.ADDTIME_MAX;
```