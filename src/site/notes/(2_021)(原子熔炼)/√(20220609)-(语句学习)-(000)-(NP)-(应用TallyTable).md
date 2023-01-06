---
{"dg-publish":true,"permalink":"/(2_021)(原子熔炼)/√(20220609)-(语句学习)-(000)-(NP)-(应用TallyTable)/"}
---


# <font color=#DC143C>(20220609)-(语句学习)-(000)-(NP)-(应用TallyTable)</font>
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

```toc
```

## P_CUT

### 00.逻辑压缩
1. 建造ROWS(01-10)
2. 步骤[1]笛卡尔积生成ROWS(001-100)
3. 步骤[2]笛卡尔积生成ROWS(001-10000)
4. 根据`@PSTRING`字符串长度通过`函数DATALENGTH`截取→生成CTETALLY
5. CTETALLY作为主表每行判断定位分隔符`SUBSTRING(@PSTRING, T.N, 1) = @PDELIMITER`→生成CTESTART
    1. 5.1返回分隔符临近下一元素的首字符`N + 1`(<strong><font color=#FF0000>函数截取的开始位置</font></strong>)
    2. 5.2通过`UNION ALL`额外插入首个元素的首字符
6. 生成CTELEN
    1. 6.1通过`函数CHARINDEX`搭配步骤[5]的CTETALLY每个元素的首位字符，找到下一个临近分隔符`CHARINDEX(@PDELIMITER, @PSTRING, S.N1)`(<strong><font color=#FF0000>函数截取的结束位置</font></strong>)
    2. 6.2`ISNULL(NULLIF(CHARINDEX(@PDELIMITER, @PSTRING, S.N1), 0) - S.N1, 8000)`→得到元素实际长度
        1. a.`函数ISNULL`搭配`函数NULLIF`处理最后一个元素分隔符缺位

### 01.表值函数本地化
```SQL
USE [odsdb]
GO
/****** Object:  UserDefinedFunction [dbo].[P_CUT]    Script Date: 2022/6/9 09:39:29 ******/
--BY徐海权
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE FUNCTION [dbo].[P_CUT](@CHARSTR VARCHAR(8000), @PARAM CHAR(1))/*避免使用(MAX DATA-TYPES)*/
RETURNS TABLE WITH SCHEMABINDING
AS RETURN
WITH
CTE_1(N) AS (SELECT 1 UNION ALL SELECT 1 UNION ALL SELECT 1 UNION ALL
             SELECT 1 UNION ALL SELECT 1 UNION ALL SELECT 1 UNION ALL
             SELECT 1 UNION ALL SELECT 1 UNION ALL SELECT 1 UNION ALL SELECT 1),
CTE_2(N) AS (SELECT 1 FROM
             CTE_1 AS a WITH(NOLOCK) CROSS JOIN CTE_1 AS b WITH(NOLOCK)),
CTE_3(N) AS (SELECT 1 FROM
             CTE_2 AS a WITH(NOLOCK) CROSS JOIN CTE_2 AS b WITH(NOLOCK)),
CTE_TALLY(N) AS (SELECT TOP (ISNULL(DATALENGTH(@CHARSTR), 0)) ROW_NUMBER()OVER(ORDER BY(SELECT NULL))
                 FROM CTE_3 WITH(NOLOCK)),
CTE_START(N_BGN) AS (SELECT 1
                     UNION ALL
                     SELECT N + 1
                     FROM CTE_TALLY WITH(NOLOCK)
                     WHERE SUBSTRING(@CHARSTR, N, 1) = @PARAM),
CTE_LEN(N_BGN, N_LEN) AS (SELECT N_BGN,
                                 ISNULL(NULLIF(CHARINDEX(@PARAM, @CHARSTR, N_BGN), 0) - N_BGN, 8888)
                          FROM CTE_START WITH(NOLOCK))
SELECT ROW_NUMBER()OVER(ORDER BY N_BGN) AS NUM,
       SUBSTRING(@CHARSTR, N_BGN, N_LEN) AS ITEM
FROM CTE_LEN WITH(NOLOCK)
```