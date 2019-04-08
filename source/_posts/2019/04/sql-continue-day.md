---
title: sql连续问题
date: 2019-04-08 19:52:11
tags: [sql]
categories: [mysql]
---



最近碰到一个sql问题：要求连续三天及以上，并且每天人流量均不少于100

```sql
mysql> select * from t_people_stat;
+----+------------+--------+
| id | start_date | people |
+----+------------+--------+
|  1 | 2019-04-03 |    100 |
|  3 | 2019-04-04 |     90 |
|  4 | 2019-04-05 |     80 |
|  5 | 2019-04-06 |    105 |
|  6 | 2019-04-07 |    109 |
|  7 | 2019-04-09 |    100 |
|  8 | 2019-04-10 |     70 |
+----+------------+--------+
7 rows in set (0.13 sec)
```

<!-- more -->

### 解决思路

+ 过滤掉人流量<100的记录
+ 生成虚拟的自增长列gid
+ 用mysql数据库中自增长的id减去虚拟的自增长的列gid,即(id-gid) as diff，构造成等差数列
+ 如果diff相同，则说明是连续的


### 实现

这里还用到了`FIND_IN_SET`和`GROUP_CONCAT`函数，*注意：GROUP_CONCAT默认最长为1024个字节*

```sql
SELECT
	tps.*
FROM
	t_people_stat tps,
(SELECT
	tps.id,
	tps.start_date,
	tps.people,
	( tps.id - tps.gid ) diff,
	GROUP_CONCAT(tps.id ORDER BY tps.id) ids,
	GROUP_CONCAT(tps.people) peoples
FROM
	( SELECT tps.*, @a := @a + 1 gid FROM t_people_stat tps, ( SELECT @a := 0 ) AS t WHERE tps.people >= 100) tps
GROUP BY diff HAVING count(id) >= 3)tpss
WHERE
FIND_IN_SET(tps.id,tpss.ids)
```


输出结果:  

```sql
+----+------------+--------+
| id | start_date | people |
+----+------------+--------+
|  5 | 2019-04-06 |    105 |
|  6 | 2019-04-07 |    109 |
|  7 | 2019-04-09 |    100 |
+----+------------+--------+
3 rows in set (0.06 sec)
```
