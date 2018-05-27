---
title: git忽略提交文件
date: 2018-05-27 11:29:28
tags: [git]
categories: [git]
---



## git忽略不想加入版本控制管理的文件

创建`.ignore`文件，与`.git/文件夹`同一级，即在项目的根目录:  

```console
.git/
  ...
  ...
.ignore
```

在`.ignore`文件中输入要过滤的文件和文件夹:  

```console
_post/
java*.md
.gitignore    #忽略.gitignore文件本身
```

<!-- more -->

## git忽略已经提交过的文件

有些时候会因为不小心或者疏忽而导致将本不需要加入版本控制的文件加入到了版本控制中，这时候修改`.ignore`文件就没有效果了

**解决方案: 先删除本地缓存，然后在`ignore`文件中填入要忽略的文件，最后再提交**

```console
git rm -r --cached xxx   //xxx表示不再想版本控制的文件
# 然后在.gitignore 文件中加入该忽略的文件
git add .
git commit -m 'update .gitignore'
```
