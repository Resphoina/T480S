---
{"dg-publish":true,"permalink":"/2-021/20220824-000-p-clone-newuser/"}
---


# <font color=#DC143C>(20220824)-(过程解析)-(000)-(数仓)-(新建账号复制权限-P_CLONE_NEWUSER)</font>
URL:: [Cloning user rights in database](https://pawlowski.cz/2011/03/31/cloning-user-rights-database/)

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

## 001.Cloning user rights in database
update: Check new post SQL Server – Cloning User Rights – [updated sp_CloneRights on GitHub](https://pawlowski.cz/2016/10/08/sql-server-cloning-user-rights-updated-sp_clonerights-on-github/)

>[!question] 问题背景
>Some times it could happen, that you need to create <strong><font color=#E6E022>a new database user</font></strong>, which will have exactly the same right as another <strong><font color=#E6E022>existing user</font></strong>.

>[!attention] 惯用方法
>In ideal scenario, you will have <strong><font color=#FF4500>all the necessary rights assigned to a database roles</font></strong>, and then when you create the new user, you simply add the user to appropriate roles to <strong><font color=#E6E022>grant all the necessary rights.</font></strong>
>
>This is ideal scenario, which is not always met, especially when you have to manage the server after somebody else, who didn’t used roles for granting rights.
>
>In such scenario a below system stored stored procedure can be very handful.

```SQL
USE [master]
GO
--============================================
-- Author:      Pavel Pawlowski
-- Created:     2010/04/16
-- Description: Copies rights of old user to new user
--==================================================
CREATE PROCEDURE sp_CloneRights (
    @oldUser sysname, --Old user from which to copy right
    @newUser sysname, --New user to which copy rights
    @printOnly bit = 1, --When 1 then only script is printed on screen, when 0 then also script is executed, when NULL, script is only executed and not printed
    @NewLoginName sysname = NULL --When a NewLogin name is provided also a creation of user is part of the final script
)
AS
BEGIN
    SET NOCOUNT ON
 
    CREATE TABLE #output (
        command nvarchar(4000)
    )
 
    DECLARE
        @command nvarchar(4000),
        @sql nvarchar(max),
        @dbName nvarchar(128),
        @msg nvarchar(max)
 
    SELECT
        @sql = N'',
        @dbName = QUOTENAME(DB_NAME())
 
    IF (NOT EXISTS(SELECT 1 FROM sys.database_principals where name = @oldUser))
    BEGIN
        SET @msg = 'Source user ' + QUOTENAME(@oldUser) + ' doesn''t exists in database ' + @dbName
        RAISERROR(@msg, 11,1)
        RETURN
    END   
 
    INSERT INTO #output(command)
    SELECT '--Database Context' AS command UNION ALL
    SELECT    'USE' + SPACE(1) + @dbName UNION ALL
    SELECT 'SET XACT_ABORT ON'
 
    IF (ISNULL(@NewLoginName, '') <> '')
    BEGIN
        SET @sql = N'USE ' + @dbName + N';
        IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = @newUser)
        BEGIN
            INSERT INTO #output(command)
            SELECT ''--Create user'' AS command
 
            INSERT INTO #output(command)
            SELECT
                ''CREATE USER '' + QUOTENAME(@NewUser) + '' FOR LOGIN '' + QUOTENAME(@NewLoginName) +
                    CASE WHEN ISNULL(default_schema_name, '''') <> '''' THEN '' WITH DEFAULT_SCHEMA = '' + QUOTENAME(dp.default_schema_name)
                        ELSE ''''
                    END AS Command
            FROM sys.database_principals dp
            INNER JOIN sys.server_principals sp ON dp.sid = sp.sid
            WHERE dp.name = @OldUser
        END'
 
        EXEC sp_executesql @sql, N'@OldUser sysname, @NewUser sysname, @NewLoginName sysname', @OldUser = @OldUser, @NewUser = @NewUser, @NewLoginName=@NewLoginName
    END
 
    INSERT INTO #output(command)
    SELECT    '--Cloning permissions from' + SPACE(1) + QUOTENAME(@OldUser) + SPACE(1) + 'to' + SPACE(1) + QUOTENAME(@NewUser)
 
    INSERT INTO #output(command)
    SELECT '--Role Memberships' AS command
 
    SET @sql = N'USE ' + @dbName + N';
    INSERT INTO #output(command)
    SELECT ''EXEC sp_addrolemember @rolename =''
        + SPACE(1) + QUOTENAME(USER_NAME(rm.role_principal_id), '''''''') + '', @membername ='' + SPACE(1) + QUOTENAME(@NewUser, '''''''') AS command
    FROM    sys.database_role_members AS rm
    WHERE    USER_NAME(rm.member_principal_id) = @OldUser
    ORDER BY rm.role_principal_id ASC'
 
    EXEC sp_executesql @sql, N'@OldUser sysname, @NewUser sysname', @OldUser = @OldUser, @NewUser = @NewUser
 
    INSERT INTO #output(command)
    SELECT '--Object Level Permissions'
 
    SET @sql = N'USE ' + @dbName + N';
    INSERT INTO #output(command)
    SELECT    CASE WHEN perm.state <> ''W'' THEN perm.state_desc ELSE ''GRANT'' END
        + SPACE(1) + perm.permission_name + SPACE(1) + ''ON '' + QUOTENAME(SCHEMA_NAME(obj.schema_id)) + ''.'' + QUOTENAME(obj.name)
        + CASE WHEN cl.column_id IS NULL THEN SPACE(0) ELSE ''('' + QUOTENAME(cl.name) + '')'' END
        + SPACE(1) + ''TO'' + SPACE(1) + QUOTENAME(@NewUser) COLLATE database_default
        + CASE WHEN perm.state <> ''W'' THEN SPACE(0) ELSE SPACE(1) + ''WITH GRANT OPTION'' END
    FROM    sys.database_permissions AS perm
        INNER JOIN
        sys.objects AS obj
        ON perm.major_id = obj.[object_id]
        INNER JOIN
        sys.database_principals AS usr
        ON perm.grantee_principal_id = usr.principal_id
        LEFT JOIN
        sys.columns AS cl
        ON cl.column_id = perm.minor_id AND cl.[object_id] = perm.major_id
    WHERE    usr.name = @OldUser
    ORDER BY perm.permission_name ASC, perm.state_desc ASC'
 
    EXEC sp_executesql @sql, N'@OldUser sysname, @NewUser sysname', @OldUser = @OldUser, @NewUser = @NewUser
 
    INSERT INTO #output(command)
    SELECT N'--Database Level Permissions'
 
    SET @sql = N'USE ' + @dbName + N';
    INSERT INTO #output(command)
    SELECT    CASE WHEN perm.state <> ''W'' THEN perm.state_desc ELSE ''GRANT'' END
        + SPACE(1) + perm.permission_name + SPACE(1)
        + SPACE(1) + ''TO'' + SPACE(1) + QUOTENAME(@NewUser) COLLATE database_default
        + CASE WHEN perm.state <> ''W'' THEN SPACE(0) ELSE SPACE(1) + ''WITH GRANT OPTION'' END
    FROM    sys.database_permissions AS perm
        INNER JOIN
        sys.database_principals AS usr
        ON perm.grantee_principal_id = usr.principal_id
    WHERE    usr.name = @OldUser
    AND    perm.major_id = 0
    ORDER BY perm.permission_name ASC, perm.state_desc ASC'
 
    EXEC sp_executesql @sql, N'@OldUser sysname, @NewUser sysname', @OldUser = @OldUser, @NewUser = @NewUser
 
    DECLARE cr CURSOR FOR
        SELECT command FROM #output
 
    OPEN cr
 
    FETCH NEXT FROM cr INTO @command
 
    SET @sql = ''
 
    WHILE @@FETCH_STATUS = 0
    BEGIN
        IF (@printOnly IS NOT NULL)
            PRINT @command
 
        SET @sql = @sql + @command + CHAR(13) + CHAR(10)
        FETCH NEXT FROM cr INTO @command
    END
 
    CLOSE cr
    DEALLOCATE cr
 
    IF (@printOnly IS NULL OR @printOnly = 0)
        EXEC (@sql)
 
    DROP TABLE #output
END
GO
EXECUTE sp_ms_marksystemobject 'dbo.sp_CloneRights'
GO
```

>[!info] 亮点
>The stored procedure allows <strong><font color=#FF4500>copying all the objects and database rights</font></strong> from the old user to a new one. <strong><font color=#FF0000>It also clones roles membership for the user.</font></strong>

+ If the `@NewLoginName` is specified then then it also creates the `@newUser` in the database for the login specified and then copies the rights.——如果`@NewLoginName`指定了，那么它还会`@newUser`在数据库中为指定的登录创建，然后复制权限。
+ `@printOnly` specifies whether the script should be printed, executed automatically or both printed and executed automatically.

As the system is marked as system, it executes in the context of the database in which is executed.

<strong><font color=#E6E022>It also allows copying rights among database roles.</font></strong> If you specify as `@oldUser` a database or application role name, then the rights of that role will be copied to the `@newUser`. <strong><font color=#FF0000>Again the `@newUser` can be a user name or database/application role name.</font></strong>

For example if you have a Integration services installed and invoke a below script
```SQL
USE [msdb]
GO
EXEC sp_CloneRights 'db_ssisadmin', 'NewUser'
```

you will receive a below script for assigning rights.

```SQL
--Database Context
USE [msdb]
SET XACT_ABORT ON
--Cloning permissions from [db_ssisadmin] to [NewUser]
--Role Memberships
--Object Level Permissions
GRANT DELETE ON [dbo].[sysssislog] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_get_dtsversion] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_make_dtspackagename] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_add_dtspackage] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_drop_dtspackage] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_reassign_dtspackageowner] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_get_dtspackage] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_addlogentry] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_listpackages] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_enum_dtspackages] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_listfolders] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_deletepackage] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_deletefolder] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_getpackage] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_getfolder] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_putpackage] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_checkexists] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_addfolder] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_renamefolder] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_setpackageroles] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_ssis_getpackageroles] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_log_dtspackage_begin] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_log_dtspackage_end] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_log_dtsstep_begin] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_log_dtsstep_end] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_log_dtstask] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_enum_dtspackagelog] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_enum_dtssteplog] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_enum_dtstasklog] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_dump_dtslog_all] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_dump_dtspackagelog] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_dump_dtssteplog] TO [NewUser]
GRANT EXECUTE ON [dbo].[sp_dump_dtstasklog] TO [NewUser]
GRANT INSERT ON [dbo].[sysssislog] TO [NewUser]
GRANT REFERENCES ON [dbo].[sysssislog] TO [NewUser]
GRANT SELECT ON [dbo].[sysssislog] TO [NewUser]
GRANT UPDATE ON [dbo].[sysssislog] TO [NewUser]
--Database Level Permissions
```

Hope you will find this script useful and hope it will save you a lot of work when cloning rights.

>[!attention] 应用
>It saved me several times, when I come to <strong><font color=#E6E022>an existing database with several hundreds of tables with rights assigned on the object level and I had to introduce a new user with exactly the same rights as an existing one.</font></strong>

One thing needs to be mentioned for end. The procedure clones only object and database right. <strong><font color=#9966CC>It doesn’t clone right for system objects, assemblies etc.</font></strong>, but you can easily extend the procedure to cover also this this.

This procedure is inspired by  a script I found in past somewhere on internet.

## 002.本地应用
### 2.0逻辑压缩
+ 创建过程——`P_CLONERIGHTS`——[[#P_CLONERIGHTS]]
+ 循环写入每一个库
+ 创建新增用户
    + `/*密码=@NEWUSER+@NEWUSER后两位*/`
+ 循环授权
    + 查阅登录账号数据库权限——`EXEC odsdb.dbo.P_SYSLOGIN_CHECK 0; /*0重新生成*/`

### 2.1[P_CLONERIGHTS]
```SQL
--USE [MASTER]
GO
/****** Object:StoredProcedure [dbo].[P_CLONERIGHTS]    Script Date:2019-04-22 15:44:53 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROC
[dbo].[P_CLONERIGHTS](@OLDUSER_INPUT SYSNAME,--复制对象
                      @NEWUSER_INPUT SYSNAME,--新建账号
                      @PRINTONLY BIT = 1,--打印还是执行
                      @NEWLOGINNAME_INPUT SYSNAME = NULL)
AS
BEGIN
SET NOCOUNT ON
--*******************************************************************************************************************************************************************
--提出人员:徐海权
--创建人员:徐海权
--创建时间:20220819
--创建用途:新建账号复制已有账号权限
--修改时间:20220822
--修改内容:修整结构
--生成结果:SELECT * FROM dbo.CloneRights_OUTPUT WITH(NOLOCK);--权限执行语句记录
--*******************************************************************************************************************************************************************
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CloneRights_OUTPUT]
IF OBJECT_ID('.dbo.CloneRights_OUTPUT', 'U') IS NOT NULL DROP TABLE dbo.CloneRights_OUTPUT;
CREATE TABLE CloneRights_OUTPUT(COMMAND NVARCHAR(4000));
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @COMMAND NVARCHAR(4000);
DECLARE @SQL NVARCHAR(MAX);
DECLARE @DBNAME NVARCHAR(128);
DECLARE @MSG NVARCHAR(MAX);
DECLARE @OLDUSER SYSNAME;
DECLARE @NEWUSER SYSNAME;
DECLARE @NEWLOGINNAME SYSNAME;
DECLARE @BREAKPOINT INT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[断点设置-赋值]
SET @BREAKPOINT = 0;/*1测试|0正式*/
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE_INPUT]
--IF @BREAKPOINT = 1
--BEGIN
--     DECLARE @OLDUSER_INPUT SYSNAME;
--     DECLARE @NEWUSER_INPUT SYSNAME;
--     DECLARE @NEWLOGINNAME_INPUT SYSNAME;
--END
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[断点设置-赋值]
--SET @OLDUSER_INPUT = 'KF';
--SET @NEWUSER_INPUT = 'KF_20220820';
--SET @NEWLOGINNAME_INPUT = 'KF_20220820';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET_INPUT]
SET @OLDUSER = @OLDUSER_INPUT;
SET @NEWUSER = @NEWUSER_INPUT;
SET @NEWLOGINNAME = @NEWLOGINNAME_INPUT;
SET @SQL = N'';
SET @DBNAME = QUOTENAME(DB_NAME());
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[判断源头账号权限是否存在]
--IF(NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = @OLDUSER))
--BEGIN
--     SET @MSG = N'SOURCE USER ' + QUOTENAME(@OLDUSER) + N' DOESN''T EXISTS IN DATABASE ' + @DBNAME;
--     RAISERROR(@MSG, 11, 1);
--     RETURN;
--END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[创建单库用户-USER]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[写入执行数据库]
INSERT INTO CloneRights_OUTPUT(COMMAND)
SELECT '--DATABASE CONTEXT' AS COMMAND
UNION ALL
SELECT 'USE' + SPACE(1) + @DBNAME
UNION ALL
SELECT 'SET XACT_ABORT ON';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[生成命令]
IF(ISNULL(@NEWLOGINNAME, '') <> '')
BEGIN
     SET @SQL = N'USE ' + @DBNAME + 
                N';IF NOT EXISTS (SELECT 1 FROM SYS.DATABASE_PRINCIPALS WHERE name = @NEWUSER)
                   BEGIN
                        INSERT INTO CloneRights_OUTPUT(COMMAND)
                        SELECT ''--CREATE USER'' AS COMMAND
                        INSERT INTO CloneRights_OUTPUT(COMMAND)
                        SELECT ''CREATE USER '' + QUOTENAME(@NEWUSER) + '' FOR LOGIN '' + QUOTENAME(@NEWLOGINNAME) +
                               CASE WHEN ISNULL(DEFAULT_SCHEMA_NAME, '''') <> '''' THEN '' WITH DEFAULT_SCHEMA = '' + QUOTENAME(dp.DEFAULT_SCHEMA_NAME)
                                    ELSE ''''
                               END AS COMMAND
                        FROM sys.database_principals dp
                        INNER JOIN sys.server_principals sp
                        ON dp.sid = sp.sid
                        WHERE dp.name = @OLDUSER
                   END';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[写入命令]
EXEC SP_EXECUTESQL
     @SQL,
     N'@OLDUSER SYSNAME, @NEWUSER SYSNAME, @NEWLOGINNAME SYSNAME',
     @OLDUSER = @OLDUSER_INPUT,
     @NEWUSER = @NEWUSER_INPUT,
     @NEWLOGINNAME = @NEWLOGINNAME_INPUT;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[读写权限-sp_addrolemember]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[插入描述]
INSERT INTO CloneRights_OUTPUT(COMMAND)
SELECT '--CLONING PERMISSIONS FROM' + SPACE(1) + QUOTENAME(@OLDUSER) + SPACE(1) + 'TO' + SPACE(1) + QUOTENAME(@NEWUSER);
INSERT INTO CloneRights_OUTPUT(COMMAND)
SELECT '--ROLE MEMBERSHIPS' AS COMMAND;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[生成命令-sp_addrolemember]
SET @SQL = N'USE ' + @DBNAME + 
           N';INSERT INTO CloneRights_OUTPUT(COMMAND)
              SELECT ''EXEC sp_addrolemember @rolename =''
                     + SPACE(1) + QUOTENAME(USER_NAME(rm.role_principal_id), '''''''')
                     + '', @membername ='' + SPACE(1) + QUOTENAME(@NEWUSER, '''''''') AS COMMAND
              FROM sys.database_role_members AS rm
              WHERE USER_NAME(rm.member_principal_id) = @OLDUSER
              ORDER BY rm.role_principal_id ASC';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[写入命令]
EXEC SP_EXECUTESQL
     @SQL,
     N'@OLDUSER sysname, @NEWUSER sysname',
     @OLDUSER = @OLDUSER_INPUT,
     @NEWUSER = @NEWUSER_INPUT;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[LEVEL PERMISSIONS]
INSERT INTO CloneRights_OUTPUT(COMMAND)
SELECT '--OBJECT LEVEL PERMISSIONS';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[生成命令-sp_addrolemember]
SET @SQL = N'USE ' + @DBNAME + 
           N';INSERT INTO CloneRights_OUTPUT(COMMAND)
              SELECT CASE WHEN perm.state <> ''W''
                          THEN perm.state_desc ELSE ''GRANT'' END + 
                     SPACE(1) + perm.permission_name + SPACE(1) + ''ON '' + QUOTENAME(SCHEMA_NAME(obj.schema_id)) + ''.'' + QUOTENAME(obj.name) + 
                     CASE WHEN cl.column_id IS NULL
                          THEN SPACE(0) ELSE ''('' + QUOTENAME(cl.name) + '')'' END + 
                     SPACE(1) + ''TO'' + SPACE(1) + QUOTENAME(@NEWUSER) COLLATE database_default + 
                     CASE WHEN perm.state <> ''W''
                          THEN SPACE(0) ELSE SPACE(1) + ''WITH GRANT OPTION'' END
              FROM sys.database_permissions AS perm
              INNER JOIN sys.objects AS obj
              ON perm.major_id = obj.[object_id]
              INNER JOIN sys.database_principals AS usr
              ON perm.grantee_principal_id = usr.principal_id
              LEFT OUTER JOIN sys.columns AS cl
              ON cl.column_id = perm.minor_id AND cl.[object_id] = perm.major_id
              WHERE usr.name = @OLDUSER
              ORDER BY perm.permission_name ASC, perm.state_desc ASC';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[写入命令]
EXEC SP_EXECUTESQL
     @SQL,
     N'@OLDUSER sysname, @NEWUSER sysname',
     @OLDUSER = @OLDUSER_INPUT,
     @NEWUSER = @NEWUSER_INPUT;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[DATABASE LEVEL PERMISSIONS]
INSERT INTO CloneRights_OUTPUT(COMMAND)
SELECT N'--DATABASE LEVEL PERMISSIONS';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[生成命令]
SET @SQL = N'USE ' + @DBNAME + 
           N';INSERT INTO CloneRights_OUTPUT(COMMAND)
              SELECT CASE WHEN perm.state <> ''W'' THEN perm.state_desc ELSE ''GRANT'' END + 
                     SPACE(1) + perm.permission_name + SPACE(1) + 
                     SPACE(1) + ''TO'' + SPACE(1) + QUOTENAME(@NEWUSER) COLLATE database_default + 
                     CASE WHEN perm.state <> ''W'' THEN SPACE(0) ELSE SPACE(1) + ''WITH GRANT OPTION'' END
              FROM sys.database_permissions AS perm
              INNER JOIN sys.database_principals AS usr
              ON perm.grantee_principal_id = usr.principal_id
              WHERE usr.name = @OLDUSER
              AND   perm.major_id = 0
              ORDER BY perm.permission_name ASC, perm.state_desc ASC';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[写入命令]
EXEC SP_EXECUTESQL
     @SQL,
     N'@OLDUSER sysname, @NEWUSER sysname',
     @OLDUSER = @OLDUSER_INPUT,
     @NEWUSER = @NEWUSER_INPUT;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[游标循环执行]
DECLARE CRLOCATION CURSOR FOR SELECT COMMAND FROM CloneRights_OUTPUT;
OPEN CRLOCATION;
FETCH NEXT FROM CRLOCATION
INTO @COMMAND;
SET @SQL = N'';
WHILE @@FETCH_STATUS = 0
BEGIN
     IF(@PRINTONLY IS NOT NULL)PRINT @COMMAND;
     SET @SQL = @SQL + @COMMAND + CHAR(13) + CHAR(10);
     FETCH NEXT FROM CRLOCATION
     INTO @COMMAND;
END;
CLOSE CRLOCATION;
DEALLOCATE CRLOCATION;
IF(@PRINTONLY IS NULL OR @PRINTONLY = 0) EXEC(@SQL);
--*******************************************************************************************************************************************************************
SET NOCOUNT OFF
END
```

### 2.2[P_CLONE_NEWUSER]
```SQL
--USE [DATABASE]
GO
/****** Object:StoredProcedure [dbo].[P_CLONE_NEWUSER]    Script Date:2019-04-22 15:44:53 ******//*[2]❆❆❆❆❆*/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROC
[dbo].[P_CLONE_NEWUSER](@NEWUSER_INPUT SYSNAME,
                        @OLDUSER_INPUT SYSNAME)
AS
BEGIN
SET NOCOUNT ON
--*******************************************************************************************************************************************************************
--提出人员:徐海权
--创建人员:徐海权
--创建时间:20220823
--创建用途:新建账号复制已有账号权限
--*******************************************************************************************************************************************************************
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[创建用户]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @SQLCMD NVARCHAR(MAX);
DECLARE @OLDUSER SYSNAME;
DECLARE @NEWUSER SYSNAME;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[赋值变量]
--SET @NEWUSER = 'CPZY-102220';
--SET @OLDUSER = 'CPZY';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[传输参数]
SET @NEWUSER = @NEWUSER_INPUT;
SET @OLDUSER = @OLDUSER_INPUT;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[生成命令]
SET @SQLCMD = 'USE MASTER;' + CHAR(13) + CHAR(10) + 
              'IF EXISTS(SELECT * FROM [master].sys.server_principals WHERE [name] = '''+ @NEWUSER + ''')' + CHAR(13) + CHAR(10) + 
              'BEGIN' + CHAR(13) + CHAR(10) + 
                    'DROP LOGIN ' + QUOTENAME(@NEWUSER) + CHAR(13) + CHAR(10) + 
              'END' + CHAR(13) + CHAR(10) + 
              'CREATE LOGIN ' + QUOTENAME(@NEWUSER) + CHAR(13) + CHAR(10) + 
              'WITH PASSWORD = ''' + LEFT(@NEWUSER, LEN(@NEWUSER) - 0) + RIGHT(@NEWUSER, 2) + ''',' + CHAR(13) + CHAR(10) + 
              'CHECK_EXPIRATION = OFF,' + CHAR(13) + CHAR(10) + 'CHECK_POLICY = OFF,' + CHAR(13) + CHAR(10) + 
              'DEFAULT_DATABASE = MASTER'
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[生成账号]
/*密码=@NEWUSER+@NEWUSER后两位*/
EXEC SP_EXECUTESQL @STMT = @SQLCMD;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[循环授权]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查阅登录账号数据库权限]
EXEC P_SYSLOGIN_CHECK 0; /*0重新生成*/
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[数据库列表]
IF OBJECT_ID('tempdb.dbo.#DATABASE_ADD', 'U') IS NOT NULL DROP TABLE #DATABASE_ADD;
SELECT [name], [dbid]
INTO #DATABASE_ADD
FROM [master].dbo.sysdatabases AS a WITH(NOLOCK)
WHERE EXISTS(SELECT *
             FROM odsdbbi.dbo.xhq_database_access AS b WITH(NOLOCK)
             WHERE b.USERNAME = @OLDUSER
             AND   a.name = b.DBNAME);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[循环执行]
WHILE EXISTS(SELECT TOP 1 * FROM #DATABASE_ADD WITH(NOLOCK))
BEGIN
     DECLARE @DATABASE VARCHAR(100);
     DECLARE @CMD NVARCHAR(MAX);
     SELECT TOP 1 @DATABASE = [NAME] FROM #DATABASE_ADD WITH(NOLOCK);
     SET @CMD = @DATABASE + '.dbo.P_CLONERIGHTS' + SPACE(1) + '''' + @OLDUSER + ''',' + SPACE(1) + 
                '''' + @NEWUSER + ''',' + SPACE(1) + '0,' + SPACE(1) + '''' + @NEWUSER + ''''
     PRINT @CMD;
     EXEC SP_EXECUTESQL @STMT = @CMD;
     DELETE FROM #DATABASE_ADD WHERE [NAME] = @DATABASE;
END;
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[检查结果]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查阅登录账号数据库权限]
EXEC P_SYSLOGIN_CHECK 0; /*0重新生成*/
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[查阅结果]
SELECT DBNAME, USERNAME, ROLENAME, IS_OWNER, IS_READER, IS_WRITER
FROM odsdbbi.dbo.xhq_database_access WITH(NOLOCK)
WHERE USERNAME IN(@OLDUSER, @NEWUSER)
ORDER BY DBNAME, USERNAME;
--*******************************************************************************************************************************************************************
SET NOCOUNT OFF
END
```