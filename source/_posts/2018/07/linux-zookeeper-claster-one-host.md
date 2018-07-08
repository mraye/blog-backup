---
title: linux中安装单机版zookeeper集群服务
date: 2018-07-08 08:44:18
tags: [cluster]
categories: [zookeeper]
---


### 创建配置文件

首先在zookeeper安装目录下的`conf`目录下创建3个配置文件`zoo1.cfg`,`zoo2.cfg`,`zoo2.cfg`,这三个配置文件差不多，只是有一点微小的差别，我的目录是`/opt/soft/zookeeper-3.4.12/conf`  


```bash
[root@localhost conf]# tree
.
├── configuration.xsl
├── log4j.properties
├── zoo1.cfg
├── zoo2.cfg
├── zoo3.cfg
├── zookeeper.out
└── zoo_sample.cfg
```
<!-- more -->

`zoo1.cfg`配置文件内容:  

```bash

tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper/zoo1 #！！区别
clientPort=2181 # zk客户端连接的端口 #！！区别

# 这里我们配置了3个服务结点
server.1=localhost:2666:3666
server.2=localhost:2667:3667
server.3=localhost:2668:3668
```


`zoo2.cfg`配置文件内容:  

```bash

tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper/zoo2 #！！区别
clientPort=2182 # zk客户端连接的端口 #！！区别

server.1=localhost:2666:3666
server.2=localhost:2667:3667
server.3=localhost:2668:3668
```


`zoo3.cfg`配置文件内容:  

```bash

tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper/zoo3 #！！区别
clientPort=2183 # zk客户端连接的端口 #！！区别

server.1=localhost:2666:3666
server.2=localhost:2667:3667
server.3=localhost:2668:3668
```

**三个配置文件最大的不同点就是`dataDir`属性和`clientPort`属性**



### 给每个服务实例设置server ID


必须给每个zookeeper服务结点设置`server id`

```bash
[root@localhost conf]# echo 1 > /tmp/zookeeper/zoo1/myid
[root@localhost conf]# echo 2 > /tmp/zookeeper/zoo2/myid
[root@localhost conf]# echo 3 > /tmp/zookeeper/zoo3/myid
```


### 启动

```bash
[root@localhost bin]# ./zkServer.sh start ../conf/zoo1.cfg
[root@localhost bin]# ./zkServer.sh start ../conf/zoo2.cfg
[root@localhost bin]# ./zkServer.sh start ../conf/zoo3.cfg
[root@localhost bin]# jps
14273 Jps
14035 QuorumPeerMain
13925 QuorumPeerMain
14159 QuorumPeerMain
```

### 客户端连接

```bash
[root@localhost bin]# ./zkCli.sh -server localhost:2181,localhost:2182,localhost:2183
WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: localhost:2181,localhost:2182,localhost:2183(CONNECTED) 1] ls /
[zookeeper]
[zk: localhost:2181,localhost:2182,localhost:2183(CONNECTED) 2] create /hello world
Created /hello
[zk: localhost:2181,localhost:2182,localhost:2183(CONNECTED) 3] ls /
[hello, zookeeper]
[zk: localhost:2181,localhost:2182,localhost:2183(CONNECTED) 4] get /hello
world
cZxid = 0x200000003
ctime = Sun Jul 08 00:08:42 UTC 2018
mZxid = 0x200000003
mtime = Sun Jul 08 00:08:42 UTC 2018
pZxid = 0x200000003
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 0
[zk: localhost:2181,localhost:2182,localhost:2183(CONNECTED) 5] quit
Quitting...
```


### 停止zookeeper服务

```bash
[root@localhost bin]# ./zkServer.sh stop ../conf/zoo1.cfg
[root@localhost bin]# ./zkServer.sh stop ../conf/zoo2.cfg
[root@localhost bin]# ./zkServer.sh stop ../conf/zoo3.cfg
[root@JARVICENAE-0A0A1881 bin]# jps
14509 Jps
```

**jps可以查看zookeeper的进程,并且以`QuorumPeerMain`名称出现**
