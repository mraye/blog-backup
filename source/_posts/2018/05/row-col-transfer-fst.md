---
title: 'mysql行列转换(一)'
date: 2018-05-26 21:25:45
tags: [sql]
categories: [mysql]
---


#### 情景一

```console
                              +----+--------+---------+
                              | id | month  | pay_fee |
                              +----+--------+---------+
                              |  1 | 201801 |      10 |
                              |  2 | 201802 |      11 |
                              |  3 | 201803 |      54 |
                              |  4 | 201804 |      32 |
                              +----+--------+---------+  
                                          |
                                          V
                      +--------+--------+--------+--------+
                      | 201801 | 201802 | 201803 | 201804 |
                      +--------+--------+--------+--------+
                      |     10 |     11 |     54 |     32 |
                      +--------+--------+--------+--------+

```

<!-- more -->

sql实现:  

```sql
SELECT
		max(CASE  WHEN month='201801' THEN pay_fee ELSE 0 end) '201801',
		max(CASE  WHEN month='201802' THEN pay_fee ELSE 0 end) '201802',
		max(CASE  WHEN month='201803' THEN pay_fee ELSE 0 end) '201803',
		max(CASE  WHEN month='201804' THEN pay_fee ELSE 0 end) '201804'
FROM
	t_month

或

SELECT
		sum(CASE  WHEN month='201801' THEN pay_fee ELSE 0 end) '201801',
		sum(CASE  WHEN month='201802' THEN pay_fee ELSE 0 end) '201802',
		sum(CASE  WHEN month='201803' THEN pay_fee ELSE 0 end) '201803',
		sum(CASE  WHEN month='201804' THEN pay_fee ELSE 0 end) '201804'
FROM
	t_month
```



#### 情景二

```console
mysql> select * from t_month;
+----+--------+---------+-----------+   
| id | month  | pay_fee | total_fee |
+----+--------+---------+-----------+
|  1 | 201801 |      10 |        20 |
|  2 | 201802 |      11 |        23 |
|  3 | 201803 |      54 |      2334 |
|  4 | 201804 |      32 |         6 |
+----+--------+---------+-----------+
                |
                V
+--------+--------+--------+--------+
| 201801 | 201802 | 201803 | 201804 |
+--------+--------+--------+--------+
|     10 |     11 |     54 |     32 |  //pay_ee
|     20 |     23 |   2334 |      6 |  //total_fee
+--------+--------+--------+--------+

```


sql实现:  

```sql
(
	SELECT
		max(CASE  WHEN month='201801' THEN pay_fee ELSE 0 end) '201801',
		max(CASE  WHEN month='201802' THEN pay_fee ELSE 0 end) '201802',
		max(CASE  WHEN month='201803' THEN pay_fee ELSE 0 end) '201803',
		max(CASE  WHEN month='201804' THEN pay_fee ELSE 0 end) '201804'
	FROM
		t_month
)
UNION
(
	SELECT
		max(CASE  WHEN month='201801' THEN total_fee ELSE 0 end) '201801',
		max(CASE  WHEN month='201802' THEN total_fee ELSE 0 end) '201802',
		max(CASE  WHEN month='201803' THEN total_fee ELSE 0 end) '201803',
		max(CASE  WHEN month='201804' THEN total_fee ELSE 0 end) '201804'
	FROM
		t_month
)
```


#### script

```sql
CREATE TABLE `t_month` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `month` varchar(255) DEFAULT NULL,
  `pay_fee` int(11) DEFAULT NULL,
  `total_fee` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8;


//插入
INSERT INTO t_month (`id`, `month`, `pay_fee`, `total_fee`)
 VALUES
('1', '201801', '10', '20'),
('2', '201802', '11', '23'),
('3', '201803', '54', '2334'),
('4', '201804', '32', '6');
```
