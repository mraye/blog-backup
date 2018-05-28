---
title: 行列转换(二)
date: 2018-05-28 22:03:36
tags: [sql]
categories: [mysql]
---


#### 情景一

1.有限列类型:


```console
+----+----------+-------+--------+
| id | fruit_id | color | weight |
+----+----------+-------+--------+
|  1 |        1 | 红    |     23 |
|  2 |        1 | 黑    |     54 |
|  3 |        2 | 红    |     12 |
|  4 |        2 | 黑    |     43 |
|  5 |        3 | 黑    |     90 |
+----+----------+-------+--------+
                |
                V

      +----------+----+----+
      | fruit_id | 红 | 黑 |
      +----------+----+----+
      |        1 | 23 | 54 |
      |        2 | 12 | 43 |
      |        3 |  0 | 90 |
      +----------+----+----+
```

<!-- more -->
sql实现:  

```sql
SELECT
   fruit_id,
	 max(case WHEN color='红' THEN weight ELSE 0 END) '红',
	 max(CASE WHEN color='黑' THEN weight ELSE 0 END) '黑'
FROM
t_fruit
GROUP BY fruit_id
```


2.无限列类型，使用动态sql(或者存储过程):  

```sql
set @sql='';

SELECT
	@sql:= concat(
		@sql,
		'sum(case color when \'',color, '\' then  weight else 0 end) as \'', color,
		'\','
	)
FROM
(
SELECT DISTINCT
	color
FROM
	t_fruit
)temp;


set @sql=concat(
	'SELECT fruit_id,',
	LEFT(@sql, CHAR_LENGTH(@sql)-1),
	-- LEFT(@sql, LENGTH(@sql)-1), 如果含有中文字符就不行...
   ' from t_fruit group by fruit_id'
);

-- SELECT @sql;
PREPARE stmt FROM @sql;
EXECUTE stmt;

```


**script**

```sql
CREATE TABLE `t_fruit` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `fruit_id` int(11) DEFAULT NULL,
  `color` varchar(255) DEFAULT NULL,
  `weight` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8;

//插入
INSERT INTO `dbl`.`t_fruit` (`id`, `fruit_id`, `color`, `weight`)
 VALUES
('1', '1', '红', '23'),
('2', '1', '黑', '54'),
('3', '2', '红', '12'),
('4', '2', '黑', '43'),
('5', '3', '黑', '90');

```



#### 情景二

列转行:  

```console
+----+----------+-------------+-----------+
| id | fruit_id | color_black | color_red |
+----+----------+-------------+-----------+
|  1 |        1 | 54          | 23        |
|  2 |        2 | 43          | 12        |
|  3 |        3 | 90          | 0         |
+----+----------+-------------+-----------+
                  |
                  V
    +----------+-------+-----------+
    | fruit_id | color | weight    |
    +----------+-------+-----------+
    |        1 | 红    | 23        |
    |        1 | 黑    | 54        |
    |        2 | 红    | 12        |
    |        2 | 黑    | 43        |
    |        3 | 红    | 0         |
    |        3 | 黑    | 90        |
    +----------+-------+-----------+
```


sql实现:  

```sql
SELECT
	fruit_id,
	'红' color,
	color_red weight
FROM
	t_color_fruit
UNION ALL
	SELECT
		fruit_id,
		'黑' color,
		color_black weight
	FROM
		t_color_fruit
ORDER BY
	fruit_id,
	color
```


**script**


```sql

CREATE TABLE `t_color_fruit` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `fruit_id` int(11) DEFAULT NULL,
  `color_black` varchar(255) DEFAULT NULL,
  `color_red` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

//插入
INSERT INTO `dbl`.`t_color_fruit` (`id`, `fruit_id`, `color_black`, `color_red`)
VALUES
('1', '1', '54', '23'),
('2', '2', '43', '12'),
('3', '3', '90', '0');

```
