---
{"dg-publish":true,"permalink":"/(2_021)(原子熔炼)/√(20211209)-(算法原子)-(000)-(NP)-(SET NOEXEC ON与RAISERROR的比较)/"}
---


# <font color=#DC143C>(20211209)-(算法原子)-(000)-(NP)-(SET NOEXEC ON与RAISERROR的比较)</font>
URL:: [How to STOP or ABORT or BREAK the execution of the statements in the current batch and in the subsequent batches separated by GO Statement based on some condition in SQL SERVER](https://sqlhints.com/2015/05/23/how-to-stop-or-abort-or-break-the-execution-of-the-statements-in-the-current-batch-and-in-the-subsequent-batches-separated-by-go-statement-based-on-some-condition-in-sql-server/)

| 萃取重点 | 萃取难点 | 萃取锚点 | 萃取输出                                                                                                                                                                                                        |
| ---- | ---- | ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| \-   | \-   | \-   | <ul><li>从结果可以看出，RETURN语句能够停止当前批处理中PRINT语句的执行，但无法停止GO语句之后的PRINT语句的执行。</li><li>如果我们使用LOG选项引发严重性 >= 20 的错误，则RAISERROR可以解决此问题。但只有具有SysAdmin权限的用户才能引发具有此严重性的错误，并且还会终止连接。由于这个原因RAISERROR不是我解决这个问题的首选方法。</li></ul> |


```ad-question
有时回过头来遇到一个场景，我需要<mark style="background: #E84A5FA6;">根据某些条件停止或中止当前批处理和后续批处理中下一个语句的执行</mark>。尝试了多种方法，但在大多数情况下能够停止当前批处理中后续语句的执行，<mark style="background: #E84A5FA6;">但不能停止go语句之后的语句</mark>。
```

两个选项有效:
+ 一个是RAISERROR
+ 一个是SET NOEXEC ON

## 让我们先了解一下我这里所说的问题是什么
### 案例RETURN
```SQL
PRINT '-----FIRST Batch - Start--------'
IF(1=1)
RETURN -- Intention is to stop execution
PRINT '-----FIRST Batch - End--------'
GO
PRINT '-----SECOND Batch--------'
GO
PRINT '-----THIRD Batch--------'
GO
```
我将RETURN语句放在第1行的意图。3是停止执行下一条PRINT语句即`PRINT '-----FIRST Batch - End--------'`和后续的PRINT语句`PRINT '-----SECOND Batch--------'`以及`PRINT '-----THIRD Batch--------'`。

### 结果
![语句01](https://sqlhints.com/wp-content/uploads/2015/05/STOP-or-ABORT-the-execution-Sql-Server-1.jpg)

萃取输出:: 从结果可以看出，RETURN语句能够停止当前批处理中PRINT语句的执行，但无法停止GO语句之后的PRINT语句的执行。

## 让我们看看用RAISERROR/THROW解决这个问题的失败尝试
### 示例1:尝试通过使用严重性级别为16的RAISERROR语句引发错误来解决
```SQL
PRINT '-----FIRST Batch - Begining--------';
IF(1 = 1)
RAISERROR('RAISERROR Approach', 16, 1);--Intention is to stop execution
PRINT '-----FIRST Batch - Ending--------';
GO
PRINT '-----SECOND Batch--------';
GO
PRINT '-----THIRD Batch--------';
GO
```
### 结果
![语句02](https://sqlhints.com/wp-content/uploads/2015/05/STOP-or-ABORT-the-execution-Sql-Server-2.jpg)

### 示例2：尝试通过使用THROW语句引发错误来解决
```SQL
PRINT '-----FIRST Batch - Begining--------';
IF (1 = 1)
THROW 50000, 'THROW Approach', 1; -- Intention is to stop execution
PRINT '-----FIRST Batch - Ending--------';
GO
PRINT '-----SECOND Batch--------';
GO
PRINT '-----THIRD Batch--------';
GO
```
### 结果
![语句03](https://sqlhints.com/wp-content/uploads/2015/05/STOP-or-ABORT-the-execution-Sql-Server-3.jpg)

从上面的例子结果可以看出，无论是RETURN语句还是RAISERROR语句或THROW语句都不能停止由GO语句分隔的后续批次中语句的执行。

## 让我们看看使用SET NOEXEC ON选项解决此问题的首选方法
```SQL
PRINT '-----FIRST Batch - Begining--------';
IF (1 = 1)
SET NOEXEC ON; -- Intention is to stop execution
PRINT '-----FIRST Batch - Ending--------';
GO
PRINT '-----SECOND Batch--------';
GO
PRINT '-----THIRD Batch--------';
GO
SET NOEXEC OFF;
```
### 结果
![语句04](https://sqlhints.com/wp-content/uploads/2015/05/STOP-or-ABORT-the-execution-Sql-Server-4.jpg)<br/>

从结果可以清楚地看出，在第3行的`SET NOEXEC ON`语句执行后，没有执行进一步的语句。您可能想知道为什么在第10行我写了`SET NOEXEC OFF`。原因是第1行的SET NOEXEC ON 语句。要为<mark style="background: #E84A5FA6;">当前会话重置此选项</mark>，我们必须执行`SET NOEXEC OFF`。

### 案例理解
```SQL
PRINT '-----FIRST Batch - Begining--------';
IF (1 = 1)
SET NOEXEC ON;
PRINT '-----FIRST Batch - Ending--------';
GO
PRINT '-----SECOND Batch--------';
GO
PRINT '-----THIRD Batch--------';
GO
SET NOEXEC OFF;
GO
PRINT '-----FOURTH Batch--------';
GO
```

### 结果
![语句05](https://sqlhints.com/wp-content/uploads/2015/05/STOP-or-ABORT-the-execution-Sql-Server-5.jpg)

请注意语句`SET NOEXEC ON`导致停止执行其后的后续语句，但它<mark style="background: #E84A5FA6;">会编译</mark>所有语句。

## 解决此问题的RAISERROR方法–不推荐
你可能想知道在本文开头我提到了RAISERROR也可以用来解决这个问题。但是本文中的一个例子表明，即使是RAISERROR也无法解决这个问题。

萃取输出:: 如果我们使用LOG选项引发严重性 >= 20 的错误，则RAISERROR可以解决此问题。但只有具有SysAdmin权限的用户才能引发具有此严重性的错误，并且还会终止连接。由于这个原因RAISERROR不是我解决这个问题的首选方法。