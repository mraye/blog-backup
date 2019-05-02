---
title: InnoDB-全文索引
date: 2019-05-02 13:44:44
tags: [sql, 索引]
categories: [mysql]
---


摘自*MySQL技术内幕-InnoDB存储引擎*


全文索引(`Full Text Search`)是将存储于数据库中的整本书或整篇文章中的任意内容信息查找出来的技术。在InnoDB1.2.x版本之前，是不支持全文检索的，而在1.2.x版本之后支持MyISAM存储引擎全文检索的全部功能



### 倒排索引

全文索引通常使用倒排索引(`inverted index`)实现。倒排索引同B+树索引一样，也是一种索引结构。它在辅助表(`auxiliary table`)中存储了单词与单词自身在一个或者多个文档中所在的 位置之间的映射

### InnoDB全文检索

InnoDB存储引擎从1.2.x版本开始支持全文检索技术，其采用 `full inverted index`的方式
