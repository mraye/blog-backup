---
title: 'git配置多个账户'
date: 2018-05-26 16:49:46
tags:
- git
---



### 创建帐户


打开git-bash,查看原有帐户:  

```bash
$ git config --list
# omit...
filter.lfs.process=git-lfs filter-process
credential.helper=manager
user.name=a***w
user.email=a***w@email.com
```

进入`~/.ssh`目录,可以看到原有的帐户默认使用`id_rsa`:  

```bash
$ ls
id_rsa  id_rsa.pub  known_hosts
```
<!-- more -->
创建新用户，并创建ssh-key,且将新产生的key值重命名为`id_rsa_aye`,不然会覆盖原帐户的ssh-key:  

```bash
$ ssh-keygen -t rsa -C  "mraye@email.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/dxjm/.ssh/id_rsa): /c/Users/dxjm/.ssh/id_rsa_aye
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /c/Users/dxjm/.ssh/id_rsa_aye.
Your public key has been saved in /c/Users/dxjm/.ssh/id_rsa_aye.pub.

$ ls
id_rsa  id_rsa.pub  id_rsa_aye  id_rsa_aye.pub  known_hosts
```

**可以看到，已经为新帐户`mraye`分别创建了公钥`id_rsa_aye.pub`和私钥`id_rsa_aye`**



### 将新产生的ssh-key交由ssh-agent管理

> ssh-agent是一个密钥管理器，运行ssh-agent以后，使用ssh-add将私钥交给ssh-agent保管，其他程序需要身份验证的时候可以将验证申请交给ssh-agent来完成整个认证过程.ssh-agent是管理多个ssh key的代理，受管理的私钥通过ssh-add来添加


因为这里会有多个ssh-key,因此把它交给ssh-agent来管理

进入bash:  

```bash
$ eval `ssh-agent -s`
Agent pid 12764

$ ssh-add ~/.ssh/id_rsa_aye
Identity added: /c/Users/dxjm/.ssh/id_rsa_aye (/c/Users/dxjm/.ssh/id_rsa_aye)
```

### 配置ssh-key与git服务器映射关系

在`~/.ssh`文件夹下创建`config`文件:  

```bash
# 该文件用于配置私钥对应的服务器
# one account
 Host github.com
 HostName github.com
 User git
 IdentityFile C:/Users/dxjm/.ssh/id_rsa

 # another account
 Host blog # github.com的别名
 HostName github.com
 User git
 IdentityFile  C:/Users/dxjm/.ssh/id_rsa_aye
```

将`id_rsa_aye.pub`文件内容填加到github帐户中的SSh管理中，那么对于mraye帐户来讲，以后项目的更新和提交都需要使用别名访问了。即，以前访问项目*git@github.com:mraye/mraye.github.io.git* ,现在就必须访问*git@blog:mraye/mraye.github.io.git*了

### 测试连通性


```bash
$ ssh -T git@github.com
Hi e***m! You've successfully authenticated, but GitHub does not provide shell access.

$ ssh -T git@blog
Hi mraye! You've successfully authenticated, but GitHub does not provide shell access.

```
