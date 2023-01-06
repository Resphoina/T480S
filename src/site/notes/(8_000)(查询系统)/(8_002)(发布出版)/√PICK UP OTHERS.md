---
{"dg-publish":true,"permalink":"/8-000/8-002/pick-up-others/"}
---


# <font color=#DC143C>(PICK UP OTHERS)</font>

```
dataview
table without id 入榜亮点, 入榜输出
where contains(TITLES, "")
```

```toc
```

## 20220817-陈浩-生产服务器访问监控
```SQL
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[228]
SELECT DISTINCT client_net_address
FROM MASTER.dbo.dbVisitInfo WITH(NOLOCK)
WHERE LEFT(loginame, 1) != '1'
AND   LEFT(loginame, 1) != '3'
AND   LEFT(loginame, 1) != '4'
AND   loginame != 'bigdata'
AND   loginame != 'store'
AND   hostname != 'HNS229'
AND   hostname != 'HNS228'
AND   updateTime IS NOT NULL
ORDER BY client_net_address;
--⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶⇶[229]
SELECT DISTINCT client_net_address
FROM DATABASE_2_29.MASTER.dbo.dbVisitInfo WITH(NOLOCK)
WHERE LEFT(loginame, 1) != '1'
AND   LEFT(loginame, 1) != '3'
AND   LEFT(loginame, 1) != '4'
AND   loginame != 'bigdata'
AND   loginame != 'store'
AND   hostname != 'HNS229'
AND   hostname != 'HNS228'
AND   updateTime IS NOT NULL
ORDER BY client_net_address;
```







