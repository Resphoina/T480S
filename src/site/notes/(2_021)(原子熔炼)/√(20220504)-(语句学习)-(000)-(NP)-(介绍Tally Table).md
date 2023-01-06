---
{"dg-publish":true,"permalink":"/(2_021)(原子熔炼)/√(20220504)-(语句学习)-(000)-(NP)-(介绍Tally Table)/"}
---


# <font color=#DC143C>(20220504)-(语句学习)-(000)-(NP)-(介绍Tally Table)</font>
URL::

```
dataview
table without id 入榜亮点, 入榜输出
where contains(TITLES, "")
```

| 萃取重点 |
| ---- |
| \-   |


```toc
```

## 01.Editor's Note
There is updated code in this article: [Tally Oh! An Improved SQL 8K CSV Splitter Function](http://www.sqlservercentral.com/articles/Tally+Table/72993/). Please use that code for production purposes and not the code in this article. The updated code is located here: [The New Splitter Functions](http://www.sqlservercentral.com/Files/The%20New%20Splitter%20Functions.zip/9510.zip).

## 02.Author UPDATE! (13 May 2011)
You'll find the makings of a "<strong><font color=#E6E022>CSV Splitter</font></strong>" in this article. Please understand that it's for the explanation of how a Tally Table can be used in place of a WHILE loop and that it is, by no stretch of imagination, an optimal solution for a "CSV Splitter. For an optional solution, please refer to the following URL ( [http://www.sqlservercentral.com/articles/Tally+Table/72993/](http://www.sqlservercentral.com/articles/Tally+Table/72993/) ) instead of using the code from this article for a "CSV Splitter" of your own.Thank you for your time.

## 03.Introduction
I actually started out writing an article on how to pass 1, 2, and 3 <strong><font color=#E6E022>dimensional "arrays" as parameters</font></strong> in stored procedures. Suddenly it dawned on me that a lot of people still have no clue what a "numbers" or "Tally" table is, never mind how it actually works.

There are dozens of things we can do in SQL that require some type of <strong><font color=#E6E022>iteration(迭代)</font></strong>. "Iteration" means "<strong><font color=#FF4500>counters(计数器)</font></strong>" and "<strong><font color=#FF4500>loops(循环)</font></strong>" to most people and <strong><font color=#FF4500>recursion(递归)</font></strong> to others. To those well familiar in the techniques of "<strong><font color=#E6E022>Set-based(基于集合)</font></strong>" programming, it means a "Numbers" or "Tally" table, instead. I like the name "Tally" table because, well, it just sounds cooler and there's no chance of anyone mistaking what I said when I say "<strong><font color=#E6E022>Tally Table</font></strong>". Everyone immediately knows what I'm talking about and what it's used for. If they don't, they always stop and ask.

So, with that in mind, we're going to explore what a Tally table is and how it works to replace loops. We'll start out simple and build to the classic example of <strong><font color=#E6E022>"splitting" a parameter</font></strong>. We'll throw in the added bonus of how to normalize an entire table worth of a CSVColumn... <strong><font color=#FF4500>all with no cursors, no loops, no functions, and no performance problems.</font></strong>

## 04.Building a Tally Table
A Tally table is nothing more than a table with a single column of <strong><font color=#FF4500>very well indexed sequential numbers</font></strong> starting at 0 or 1 (mine start at 1) and going up to some number. The largest number in the Tally table should not be just some arbitrary choice. It should be based on what you think you'll use it for. I split <strong><font color=#E6E022>VARCHAR(8000)'s</font></strong> with mine, so it has to be at least 8000 numbers. Since I occasionally need to generate 30 years of dates, I keep most of my production Tally tables at 11,000 or more which is more than 365.25 days times 30 years.

There are many methods to build a Tally table. Let's use one of the most obvious, a <strong><font color=#E6E022>loop</font></strong>. Why? Because it's familiar ground for a lot of folks... trust me for a minute... I have other points to make in this article...

### 0401.循环方式创建脚本
```SQL
USE TEMPDB;--DB that everyone has where we can cause no harm
SET NOCOUNT ON;--Supress the auto-display of rowcounts for appearance/speed
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @STARTTIME DATETIME;--Timer to measure total duration
DECLARE @COUNTER INT;--Create and preset a loop counter
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @STARTTIME = GETDATE();--Start the timer
SET @COUNTER = 1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[Create and populate a Tally table]
--Conditionally drop and create the table/Primary Key
IF OBJECT_ID('dbo.Tally') IS NOT NULL DROP TABLE dbo.Tally;
CREATE TABLE dbo.Tally(N INT, CONSTRAINT PK_Tally_N PRIMARY KEY CLUSTERED(N));
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[Populate the table using the loop and counter]
WHILE @Counter <= 11000
BEGIN
     INSERT INTO dbo.Tally(N)VALUES(@Counter);
     SET @Counter = @Counter + 1;
END;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[Display the total duration]
SELECT STR(DATEDIFF(ms, @StartTime, GETDATE())) + ' Milliseconds duration';
```
^5kz5em

That will take about <strong><font color=#70f3ff>750 milliseconds</font></strong> to generate. Like I said, there are many ways to generate a Tally table. Let me introduce you to my favorite way and then I'll explain why it's a favorite...

### 0402.系统表方式创建脚本
```SQL
USE tempdb;--DB that everyone has where we can cause no harm
SET NOCOUNT ON;--Supress the auto-display of rowcounts for appearance/speed
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @STARTTIME DATETIME;--Timer to measure total duration
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @STARTTIME = GETDATE();--Start the timer
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[Create and populate a Tally table]
IF OBJECT_ID('dbo.Tally') IS NOT NULL DROP TABLE dbo.Tally;--Conditionally drop
--Create and populate the Tally table on the fly
SELECT TOP 11000--equates to more than 30 years of dates
       IDENTITY(INT, 1, 1) AS N
INTO dbo.Tally
FROM master.dbo.syscolumns AS sc1,
     master.dbo.syscolumns AS sc2;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[Add a Primary Key to maximize performance]
ALTER TABLE dbo.Tally ADD CONSTRAINT PK_Tally_N PRIMARY KEY CLUSTERED(N)WITH FILLFACTOR = 100;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[Let the public use it]
GRANT SELECT, REFERENCES ON dbo.Tally TO PUBLIC;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[Display the total duration]
SELECT STR(DATEDIFF(MS, @STARTTIME, GETDATE())) + ' MILLISECONDS DURATION';
```
^9taie2

The first time you run it, it'll take about 220 milliseconds so, right up front, <strong><font color=#E6E022>it beats the loop</font></strong>. <strong><font color=#E6E022>Run it a second time and you'll find that it takes less time</font></strong>... anywhere from 73 to 186 milliseconds depending on what the system is doing.

<strong><font color=#E6E022>The key here is that we've already replaced one loop using an IDENTITY and a CROSS JOIN.</font></strong> Cross Joins can be a real friend because they can be used in place of loops and, on a properly indexed table like a Tally table, they run nasty fast. <strong><font color=#FF0000>The reason why this method IS my favorite is because of the Cross Join</font></strong>... <strong><font color=#FF4500>it makes the code very short</font></strong>, very easy to remember, and very fast. We'll see another example of a Cross Join actually using the Tally table later in the article.

Don't delete the Tally table we just built. We're going to play with it to show you how it works.

## 05.Direct Loop Replacement as a COUNTER
<strong><font color=#FF4500>Right after the "Hello World" problem, most programming language instructors teach how to "Loop".</font></strong> The problem normally manifests itself as "Produce and display a count from 1 to 10. In SQL Server, that frequently (and, unfortunately) is demonstrated as the following...

```SQL
--===============================================
--      Display the count from 1 to 10
--      using a loop.
--===============================================
--===== Declare a counter
DECLARE @N INT
    SET @N = 1
--===== While the counter is less than 10...
  WHILE @N <= 10
  BEGIN
        --==== Display the count
        SELECT @N
        --==== Increment the counter
        SET @N = @N + 1
    END
```

This works because we do iterations using a counter. Let me repeat, we do iterations using a _counter_. This can be defined as "For each count from 1 to 10, display the value of the count".

<strong><font color=#800080>萃取重点::</font></strong><strong><font color=#70f3ff>Also notice, the SELECT was executed 10 times and that creates 10 different result sets. That's NOT set based programming.</font></strong> That's what I refer to as "<strong><font color=#FF0000>RBAR</font></strong>" (pronounced "ree-bar" and is a "Modenism" for "<strong><font color=#FF0000>Row By Agonizing Row</font></strong>"). Let's see how to do that with a Tally table...

```SQL
--===============================================
--      Display the count from 1 to 10
--      using a Tally table.
--===============================================
 SELECT N
   FROM dbo.Tally
  WHERE N <= 10
  ORDER BY N
```

This works because we use existing rows to do the "iterations". Let me repeat, <strong><font color=#FF4500>we do the iterations using _existing rows_</font></strong>. This can be defined as "For each row from 1 to 10, display the value of the row". The BIG difference is that we only use a single SELECT and we don't actually have to count as we go. We just limit how many rows we use and what the values of the rows are. It produces the same result as the loop does (count of 1 to 10), but it does it using a single SELECT. <strong><font color=#FF0000>In other words, it produces a single result set with the entire answer.</font></strong> Nothing RBAR about that... that's set based programming.

## 06.Stepping Through Characters
Let's say we have a parameter (variable) that looks like this (note the starting and ending commas)...

```SQL
--===== Simulate a passed parameter
DECLARE @Parameter VARCHAR(8000)
    SET @Parameter = ',Element01,Element02,Element03,Element04, Element05,'
```

Let's write a loop just to show the character position for each character in the order they appear. Like this...

```SQL
--===== Simulate a passed parameter
DECLARE @PARAMETER VARCHAR(8000);
SET @PARAMETER = ',Element01,Element02,Element03,Element04,Element05,';
--===== Declare a character counter
DECLARE @N INT;
SET @N = 1;
--===== While the character counter is less then the length of the string
WHILE @N <= LEN(@Parameter)
BEGIN
     --==== Display the character counter and the character at that position
     SELECT @N, SUBSTRING(@Parameter, @N, 1);
     --==== Increment the character counter
     SET @N = @N + 1;
END;
```

Note that as the loop progresses, the character counter "steps" through the string using <strong><font color=#E6E022>SUBSTRING</font></strong> to display each character. It starts at "1" and ends when it gets to the end of the string displaying a character at that particular position for each iteration of the loop. Note also that it creates 51 separate result sets. That means that 51 individual SELECTs were executed. Heh... that qualifies as "RBAR"!

Editor's Note: There is updated code in this article: [Tally Oh! An Improved SQL 8K CSV Splitter Function](http://www.sqlservercentral.com/articles/Tally+Table/72993/). Please use that code for production purposes and not the code in this article. The updated code is located here: [The New Splitter Functions](http://www.sqlservercentral.com/Files/The%20New%20Splitter%20Functions.zip/9510.zip).

Let's see how to do it with the Tally table...

```SQL
--===== Simulate a passed parameter
DECLARE @PARAMETER VARCHAR(8000);
SET @PARAMETER = ',Element01,Element02,Element03,Element04,Element05,';
--===== Do the same thing as the loop did... "Step" through the variable
--      and return the character position and the character...
SELECT N, SUBSTRING(@Parameter, N, 1)
FROM dbo.Tally
WHERE N <= LEN(@Parameter)
ORDER BY N;
```

Just like counting from 1 to 10, <strong><font color=#E6E022>both the loop and the Tally table count from 1 to the length of the parameter</font></strong>. The Tally table is a direct replacement for the loop. Look at the following graphic. Both the loop and the Tally table do exactly the same thing except the Tally table only uses 1 SELECT and returns a single result set. The rows of the Tally table act as the counter except it's set based.

![a direct replacement for the loop|L](https://www.sqlservercentral.com/wp-content/uploads/legacy/f9b42adb1730fac82374f5c1607161297c997ca6/931.jpg)

## 07."Jumping" to Certain Characters
Let's do almost the same thing, but let's just return the positions of the commas. First, one method to do it using a loop...

```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[Simulate a passed parameter]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @PARAMETER VARCHAR(8000);
DECLARE @N INT;--Declare a charaction counter
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @PARAMETER = ',Element01,Element02,Element03,Element04,Element05,';
SET @N = 1;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[WHILE]
WHILE @N <= LEN(@PARAMETER)
--While the character counter is less then the length of the string
BEGIN
     --Display the character counter and the character at that position but only if a comma is present at that position.
     IF SUBSTRING(@PARAMETER, @N, 1) = ','
     BEGIN
          SELECT @N, SUBSTRING(@PARAMETER, @N, 1);
     END;
     SET @N = @N + 1;--Increment the character counter
END;
```

Did you notice how the loop is slowing down? Now, let's try the same thing with the Tally table...

```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[Simulate a passed parameter]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DECLARE]
DECLARE @PARAMETER VARCHAR(8000);
SET @PARAMETER = ',Element01,Element02,Element03,Element04,Element05,';
--Do the same thing as the loop did... "Step" through the variable and return the character position and the character...
--but only if it's a comma in that position...
SELECT N, SUBSTRING(@PARAMETER, N, 1)
FROM dbo.Tally
WHERE N <= LEN(@PARAMETER)
AND   SUBSTRING(@PARAMETER, N, 1) = ','
ORDER BY N;
```

Notice, that almost didn't slow down at all. It's doing the same as the loop solution. <strong><font color=#E6E022>The big difference is that it's doing it all in a single SELECT.</font></strong> <strong><font color=#FF4500>It's using the existing rows in the Tally table as a counter.</font></strong> In essence, it' joining to the parameter at the character level... <strong><font color=#E6E022>and that makes it very, very fast especially since the values of the Tally table are cached</font></strong> and the counters in the loop are not.

## 08.Going all the way... do the "Split".
Ok, we now understand that a Tally table is a direct replacement for some loops... especially loops that count. We've also seen that the Tally table can single out certain characters much faster than using a loop. MUCH faster. Let's go all the way... let's split the parameter we've been using into the individual elements that are marked by the commas. Here's one way to do it with a loop except this time, I've cheated a bit to help the loop find the comma's using CHARINDEX instead of stepping through each character just to give it a fighting chance. I've also made the parameter more like what you'd get from a GUI... no leading or trailing commas...

> [!attention]
> 1. ADD START AND END COMMAS TO THE PARAMETER SO WE CAN HANDLE SINGLE ELEMENTS——补全分割符号
> 2. DO THE INSERT USING THE VALUE BETWEEN THE COMMAS——捕捉分割两个符号相夹部分

```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[Simulate a passed parameter]
DECLARE @PARAMETER VARCHAR(8000);
DECLARE @N INT;--DECALRE A VARIABLE TO REMEMBER THE POSITION OF THE CURRENT COMMA
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @PARAMETER = 'ELEMENT01,ELEMENT02,ELEMENT03,ELEMENT04,ELEMENT05';
SET @PARAMETER = ',' + @PARAMETER + ',';--ADD START AND END COMMAS TO THE PARAMETER SO WE CAN HANDLE SINGLE ELEMENTS
SET @N = 1;--PREASSIGN THE CURRENT COMMA AS THE FIRST CHARACTER
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CREATE A TABLE TO STORE THE RESULTS IN]
DECLARE @ELEMENTS TABLE(NUMBER INT IDENTITY(1, 1),--ORDER IT APPEARS IN ORIGINAL STRING
                        VALUE VARCHAR(8000));--THE STRING VALUE OF THE ELEMENT
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[LOOP THROUGH AND FIND EACH COMMA, THEN INSERT THE STRING VALUE]
                        --FOUND BETWEEN THE CURRENT COMMA AND THE NEXT COMMA.  @N IS THE POSITION OF THE CURRENT COMMA.
WHILE @N < LEN(@PARAMETER)--DON'T INCLUDE THE LAST COMMA
BEGIN
     INSERT INTO @ELEMENTS--DO THE INSERT USING THE VALUE BETWEEN THE COMMAS
     VALUES(SUBSTRING(@PARAMETER, @N + 1, CHARINDEX(',', @PARAMETER, @N + 1) - @N - 1));
     --FIND THE NEXT COMMA
     SELECT @N = CHARINDEX(',', @PARAMETER, @N + 1);
END;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[READ]
SELECT * FROM @ELEMENTS;
```

Again, all this does is find a comma and "remembers" its position. <strong><font color=#E6E022>Then it uses CharIndex to find the next comma and inserts what's between the commas into a table variable.</font></strong> It quits looping when it runs out of commas.

Let's try the same thing with the Tally table...

```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SIMULATE A PASSED PARAMETER]
DECLARE @PARAMETER VARCHAR(8000);
--CREATE A TABLE TO STORE THE RESULTS IN
DECLARE @ELEMENTS TABLE(NUMBER INT IDENTITY(1, 1),--ORDER IT APPEARS IN ORIGINAL STRING
                        VALUE VARCHAR(8000));--THE STRING VALUE OF THE ELEMENT
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SET @PARAMETER = 'ELEMENT01,ELEMENT02,ELEMENT03,ELEMENT04,ELEMENT05';
SET @PARAMETER = ',' + @PARAMETER + ',';--ADD START AND END COMMAS TO THE PARAMETER SO WE CAN HANDLE SINGLE ELEMENTS
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[INSERT]
--JOIN THE TALLY TABLE TO THE STRING AT THE CHARACTER LEVEL AND
--WHEN WE FIND A COMMA, INSERT WHAT'S BETWEEN THAT COMMAND AND THE NEXT COMMA INTO THE ELEMENTS TABLE
INSERT INTO @ELEMENTS(VALUE)
SELECT SUBSTRING(@PARAMETER, N + 1, CHARINDEX(',', @PARAMETER, N + 1) - N - 1)
FROM dbo.TALLY WITH(NOLOCK)
WHERE N < LEN(@PARAMETER)
AND   SUBSTRING(@PARAMETER, N, 1) = ',';--NOTICE HOW WE FIND THE COMMA
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[RESULT]
SELECT * FROM @ELEMENTS;
```

The details are in the comments. What I want to point out is that they're both very fast. 5 elements just isn't enough to determine a winner here. Let's make a "monster" parameter of 796 elements (last one will be blank) and see which one wins... here's the full code I ran...

### LOOP METHOD vs TALLY TABLE METHOD
```SQL
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[LOOP METHOD]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SIMULATE A PASSED PARAMETER]
DECLARE @PARAMETER VARCHAR(8000);
DECLARE @N INT;--DECALRE A VARIABLE TO REMEMBER THE POSITION OF THE CURRENT COMMA
--CREATE A TABLE TO STORE THE RESULTS IN
DECLARE @ELEMENTS TABLE(NUMBER INT IDENTITY(1, 1), --ORDER IT APPEARS IN ORIGINAL STRING
                        VALUE VARCHAR(8000));--THE STRING VALUE OF THE ELEMENT
SET @PARAMETER = REPLICATE('ELEMENT01,ELEMENT02,ELEMENT03,ELEMENT04,ELEMENT05,', 159);
SET @PARAMETER = ',' + @PARAMETER + ',';--ADD START AND END COMMAS TO THE PARAMETER SO WE CAN HANDLE SINGLE ELEMENTS
SET @N = 1;--PREASSIGN THE CURRENT COMMA AS THE FIRST CHARACTER
SET NOCOUNT ON;
WHILE @N < LEN(@PARAMETER)--DON'T INCLUDE THE LAST COMMA
--LOOP THROUGH AND FIND EACH COMMA, THEN INSERT THE STRING VALUE  FOUND BETWEEN THE CURRENT COMMA AND THE NEXT COMMA.  @N IS
--THE POSITION OF THE CURRENT COMMA.
BEGIN
     --DO THE INSERT USING THE VALUE BETWEEN THE COMMAS
     INSERT INTO @ELEMENTS
     VALUES(SUBSTRING(@PARAMETER, @N + 1, CHARINDEX(',', @PARAMETER, @N + 1) - @N - 1));
     --FIND THE NEXT COMMA
     SELECT @N = CHARINDEX(',', @PARAMETER, @N + 1);
END;
GO
--↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹↹[TALLY TABLE METHOD]
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SIMULATE A PASSED PARAMETER]
DECLARE @PARAMETER VARCHAR(8000)
--CREATE A TABLE TO STORE THE RESULTS IN
DECLARE @ELEMENTS TABLE(NUMBER INT IDENTITY(1, 1),--ORDER IT APPEARS IN ORIGINAL STRING;
                        VALUE VARCHAR(8000));--THE STRING VALUE OF THE ELEMENT
SET @PARAMETER = REPLICATE('ELEMENT01,ELEMENT02,ELEMENT03,ELEMENT04,ELEMENT05,', 159);
SET @PARAMETER = ',' + @PARAMETER + ',';--ADD START AND END COMMAS TO THE PARAMETER SO WE CAN HANDLE SINGLE ELEMENTS
SET NOCOUNT ON;
--JOIN THE TALLY TABLE TO THE STRING AT THE CHARACTER LEVEL AND
--WHEN WE FIND A COMMA, INSERT WHAT'S BETWEEN THAT COMMAND AND THE NEXT COMMA INTO THE ELEMENTS TABLE
INSERT INTO @ELEMENTS(VALUE)
SELECT SUBSTRING(@PARAMETER, N + 1, CHARINDEX(',', @PARAMETER, N + 1) - N - 1)
FROM dbo.TALLY
WHERE N < LEN(@PARAMETER)
AND   SUBSTRING(@PARAMETER, N, 1) = ',';--NOTICE HOW WE FIND THE COMMA
```

And, here's the results according to the Profiler...

![LOOP METHOD vs TALLY TABLE METHOD|L](https://www.sqlservercentral.com/wp-content/uploads/legacy/d9272a835f94c71d70801ed522746fd047cc6e3f/932.jpg)

<strong><font color=#E6E022>The Tally table method wins for Duration, CPU time, and RowCounts. It's looses in Reads</font></strong> and that bothers some folks, but it shouldn't. Part of the reason the Tally table method works so well is because parts of the Tally table get cached and those are "Logical Reads" from _memory_. If you think these differences are small, multiply them by 10,000 or a million and see how much difference there is. If you want performance, stop using loops and start using a Tally table.

## 09.One Final "Split" Trick with the Tally Table
You've seen it... some poor slob posts that <strong><font color=#70f3ff>(s)he has a table and it has a CSV column in it.</font></strong> "How do you join to it?", they ask. The correct answer, of course, is to normalize the table so that there is no CSV column and everyone says so on the post. Heh, but then the recommendation that follows is usually "<strong><font color=#70f3ff>Write a cursor to normalize the table.</font></strong>"

The Tally table can normalize the whole table in one simple select... and remember when I said that <strong><font color=#FF0000>I liked Cross Joins on the Tally table because it makes code short and fast?</font></strong> Check this out...

![|L|600](https://raw.githubusercontent.com/Resphoina/MDPIC/master/markdown2022-05-19-1011-MD-%E6%95%B0%E6%8D%AE%E5%88%86%E6%9E%90-Tally%E6%8B%86%E5%88%86CSV.jpg)

```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[CREATE A SAMPLE DENORMALIZED TABLE WITH A CSV COLUMN]
IF OBJECT_ID('tempdb.dbo.#MyHead', 'U') IS NOT NULL DROP TABLE #MyHead;
CREATE TABLE #MyHead(PK INT IDENTITY(1, 1) PRIMARY KEY CLUSTERED,
                     CsvColumn VARCHAR(500));
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[INSERT]
INSERT INTO #MyHead
SELECT '1,5,3,7,8,2'
UNION ALL
SELECT '7,2,3,7,1,2,2'
UNION ALL
SELECT '4,7,5'
UNION ALL
SELECT '1'
UNION ALL
SELECT '5'
UNION ALL
SELECT '2,6'
UNION ALL
SELECT '1,2,3,4,55,6';
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SPLIT OR "NORMALIZE" THE WHOLE TABLE AT ONCE]
SELECT mh.PK,
       SUBSTRING(',' + mh.CsvColumn + ',', N + 1, CHARINDEX(',', ',' + mh.CsvColumn + ',', N + 1) - N - 1) AS VALUE
FROM dbo.Tally AS t WITH(NOLOCK)--主表
CROSS JOIN #MyHead AS mh WITH(NOLOCK)
WHERE N < LEN(',' + mh.CsvColumn + ',')
AND   SUBSTRING(',' + mh.CsvColumn + ',', N, 1) = ',';--逢分隔符处理
```

No cursor, no loops, no functions, no performance problems... all done with the great loop substitute, the Tally Table.

## 10.Dozens of Other Uses
<strong><font color=#E6E022>Once you've made the realization that joining to a Tally table is like creating a loop</font></strong>, you can do dozens of other things. Need to create a derived table with all the dates in a date range so you can outer join to it and get the SUM of 0 for dates you have no data for? That's easy with a Tally table...

```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[PRESETS]
USE NORTHWIND;
DECLARE @DATESTART DATETIME;
DECLARE @DATEEND DATETIME;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[FIND THE MIN AND MAX DATES IN THE RANGE OF DATA]
SELECT @DATESTART = MIN(SHIPPEDDATE),
       @DATEEND = MAX(SHIPPEDDATE)
FROM dbo.ORDERS WITH(NOLOCK);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[FIND THE TOTAL FREIGHT FOR EACH DAY EVEN IF THE DAY HAD NO SHIPMENTS]
SELECT DATES.SHIPPEDDATE, ISNULL(SUM(O.FREIGHT), 0) AS TOTALFREIGHT
FROM dbo.ORDERS AS O WITH(NOLOCK)
RIGHT OUTER JOIN(SELECT T.N - 1 + @DATESTART AS SHIPPEDDATE--CREATE ALL SHIPPED DATES BETWEEN START AND END DATE
                 FROM dbo.TALLY AS T WITH(NOLOCK)
                 WHERE T.N - 1 + @DATESTART <= @DATEEND) DATES
ON O.SHIPPEDDATE = DATES.SHIPPEDDATE
GROUP BY DATES.SHIPPEDDATE;
```

How about making a "shift" table with 3 shifts per day starting on '2008-01-01 06:00' for the next 5 years... again, easy with the Tally table...

```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[PRESETS]
DECLARE @DATESTART DATETIME;
DECLARE @DATEEND DATETIME;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[SET]
SELECT @DATESTART = '2008-01-01 06:00';
SELECT @DATEEND = DATEADD(YY, 5, @DATESTART);
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[DISPLAY THE SHIFT NUMBER AND DATE/TIMES]
SELECT (T.N - 1) % 3 + 1 AS SHIFTNUMBER,
       DATEADD(HH, 8 * (T.N - 1), @DATESTART) AS SHIFTSTART,
       DATEADD(HH, 8 * (T.N), @DATESTART) AS SHIFTEND
FROM dbo.TALLY AS T WITH(NOLOCK)
WHERE DATEADD(HH, 8 * (T.N - 1), @DATESTART) <= @DATEEND;
```

Notice that the "%3" is used to provide the shift number based on the value of the Tally table. We're also making dates by multiplying the Tally table value by 8 hours which generates the dates/times from the base date. Like I said, there are dozens of uses... Just do a search on "Numbers Table" or "Tally Table".

## 11.Conclusion
<strong><font color=#E6E022>This article isn't meant to show you every thing you can use a Tally table for.</font></strong> Rather, it was an introduction as to what a Tally table is and how it actually works to replace loops in a set based fashion. In many cases, the Tally table is a direct replacement for a loop that counts. In other cases, something a bit more exotic needs to happen in the form of a Cross Join with the Tally table like when you want to split a whole column of CSV's. I think that you'll find that no matter what the use is, a Tally table will always beat the pants off of a looping solution.

This article hasn't even scratched the surface... there are dozens of uses for this simple, yet mighty tool. Many folks have posted some pretty interesting functions have to do with "Numbers" and "Tally" tables. Just do a search for "Numbers Table" or "Tally Table". You'll be amazed.

What I hope you take from this article is that now you know just exactly how a Tally table works. With that fundamental understanding, you'll be able to easily read the intent of code that uses a Tally table and easily identify "tuning/performance opportunities" in existing and new code.

Thanks for listening, folks.

--Jeff Moden

_"RBAR is pronounced "ree-bar" and is a "Modenism" for "**R**ow-**B**y-**A**gonizing-**R**ow"_