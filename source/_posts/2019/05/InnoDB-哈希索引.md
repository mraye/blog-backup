---
title: InnoDB-哈希索引
date: 2019-05-02 13:44:30
tags: [sql, 索引]
categories: [mysql]
---


摘自*MySQL技术内幕-InnoDB存储引擎*


InnoDB存储引擎使用哈希算法对字典进行查找，其冲突机制采用链表方式，哈希函数采用除法散列方式



### 自适应哈希索引

自适应哈希索引采用哈希表的方式实现。不同的是，这仅是数据库自身创建并使用的，DBA本身并不能对其进行干预。自适应哈希索引经哈希函数映射到一个哈希表中，因此对字典类型的查找非常快速，如`select * from table where col='tom'`,但对于范围查找就无能为力了
