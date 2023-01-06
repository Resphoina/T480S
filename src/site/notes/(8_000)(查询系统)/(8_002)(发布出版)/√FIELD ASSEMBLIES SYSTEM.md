---
{"dg-publish":true,"permalink":"/(8_000)(查询系统)/(8_002)(发布出版)/√FIELD ASSEMBLIES SYSTEM/"}
---


# <font color=#DC143C>(FIELD ASSEMBLIES)</font>
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

## 大表修改表结构操作规范
1. 停止数据更新并新建一张表
2. 数据导入至新表
3. 建立索引
4. 与原表表名互换
5. 核对数据无异常
6. 恢复数据更新
7. 观察至少1天确认无问题后，删除原表或迁移原表数据到TEMP库(备份)

## 大表数据更新缓慢解决方案
1. `NOT EXISTS`找出原表数据没有更新的部分，写入新表
2. 过程中计算生成或拉取生产的更新数据`INSERT INTO`上一步骤的新表
3. 检查新表的数据量是否异常
4. 新表建立索引
5. 与原表表名互换
6. 更新过程下次执行时删除互换的旧表

## 项目实施过程
1. 业务需求调研
2. 数据清理
3. 指标体系梳理
4. 数据模型构建

## 数据库删库视图引用检查
### 20220730版本
+ 1——主操作库`ODSDBXXX`和`ODSDBFQXXX`，右键选择→任务→生成脚本→导出视图
+ 2——检查具体哪些视图包含目标库数据表
+ 3——逐个修改视图
+ 4——上下游检查过程脚本引用情况进行修改(在线文档公告)
+ 5——再次导出视图检查确认步骤[1]修改
+ 6——执行更新上下游关系表检查确认步骤[4]修改

## 20220802——如何做好一个BI项目的规划和需求定义
[[(2_022)(数据质量)/√(20220802)-(数据质量)-(222)-(方法)-(如何做好一个BI项目的规划和需求定义)|如何做好一个BI项目的规划和需求定义]]

## 20220803——脚本未完信息记录
1. 任务描述:
2. 任务类型:
3. 后续操作:
4. 服务对象:
5. 保存时间:

## 20220817——批量授予非管理员作业权限
1. 禁用作业
    1. `[(00)(徐海权)【仓库管理】【权限监控】作业权限开通回收(每20分钟)-【RX】]`
2. 执行作业
    1. `[(00)(徐海权)【仓库管理】【作业监控】作业基础信息扫描(0420|1320|1515)-【RX】]`
    2. 找到作业——[[#脚本代码-找到作业]]
    3. 脚本启动
        1. `EXEC msdb.dbo.sp_start_job @JOB_ID = '2C04C6C8-E90E-46C7-A9F2-E9F466D7E119';`
3. 回收作业权限
    1. `USE odsdbfq; EXEC p_sysjob_access 0, 0, 'sa';/*回收*/`
4. 插入需要授予明细——[[#脚本代码-插入需要授予明细]]
5. 执行作业
    1. `USE odsdbfq; EXEC p_sysjob_access 1, 0, 'sa';/*执行*/`

### 脚本代码-找到作业
```SQL
SELECT JOB_ID
FROM dbo.MRX_SYSJOBS_INFO WITH(NOLOCK)
WHERE CHARINDEX('作业基础信息扫描', JOBNAME, 1) > 1;
```

### 脚本代码-插入需要授予明细
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[插入需要授予明细]
INSERT INTO odsdbfqbi.dbo.job_update_insert(ZhangHao, ZuoYeMingCheng)
SELECT b.userid,
       a.name
FROM dbo.XTB_sysjobs_owner_output AS a WITH(NOLOCK)
INNER JOIN dbo.user_dept_detail   AS b WITH(NOLOCK)
ON b.username = a.job_owner
WHERE b.stat = '在职'
AND   ISNULL(b.dept4, '') != '信息总部'
AND   b.dept3 != '信息总部'
AND (b.position  LIKE '%分析%' OR b.position  LIKE '%IT%' OR b.position  LIKE '%系统%')
ORDER BY JOB_OWNER;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[确认查阅授予明细]
SELECT ZhangHao, ZuoYeMingCheng, ZuoYeID, DealStatus, UPDATETIME
FROM odsdbfqbi.dbo.job_update_insert WITH(NOLOCK)
WHERE UPDATETIME >= CONVERT(CHAR(8), CURRENT_TIMESTAMP, 112)
ORDER BY UPDATETIME DESC;
```

```SQL
月度经营分析
月度绩效考核
年度关键考核指标
日常管理分析

```