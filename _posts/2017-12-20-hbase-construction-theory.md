---
layout: post
title: "HBASE原理与架构"
description: "HBASE原理与架构"
categories: [database]
tags: [hbase]
redirect_from:
  - /2017/12/20/
---
HBASE是一个构建在HDFS上的分布式列存储系统，比较起传统的关系型数据库的优点在于，可以实现高性能的并发读写操作。同时HBase不同于传统关系型数据库的面向行存储，HBase是面向列存储，列可以动态增加，列为空就不存储数据，能更好的节省存储空间。Hbase还会对数据进行透明切分，使得存储本身具有了水平伸缩性

## HBASE特点
- 数据量大:一个表可以有数十亿行，上百万列
- 无模式:每行都有一个可排序的主键和任意多的列，列可以根据需要动态的增加，同一张表中不同的行可以有截然不同的列
- 面向列：面向列（族）的存储和权限控制，列（族）独立检索
- 稀疏:空（null）列并不占用存储空间，表可以设计的非常稀疏； 
- 数据多版本:每个单元中的数据可以有多个版本，默认情况下版本号自动分配，是单元格插入时的时间戳
- 数据类型单一:Hbase中的数据都是字符串，没有类型。

## HBASE数据模型
<center><img src="/assets/images/post/2017-12-20-hbase-construction-theory/hbase-data-model.png" width="500px"/></center>

- 行健(Rowkey):是Byte array，是表中每条记录的“主键”，方便快速查找，Rowkey的设计非常重要(检索方式)
- 版本(version):每条记录可动态添加Version Number，类型为Long，默认值是系统时间戳，可由用户自定义
- 列族(Column Family):拥有一个名称，包含一个或者多个相关列
- 列(Colume):属于某一个Column Family，格式:familyName:columnName
- 值(value)

Table在水平方向有一个或者多个Column Family组成，一个Column Family中可以由任意多个Column组成，即Column Family支持动态扩展，无需预先定义Column的数量以及类型，所有Column均以二进制格式存储，用户需要自行进行类型转换

## HBASE物理模型

### 物理存储

- Table中所有行都按照row key的字典序排列
- Table在行的方向上分割为多个Region(相当于一个region对应某张表的startRowkey到endRowKey的数据)
- Region按大小分割的，每个表开始只有一个region，随着数据增多，region不断增大，当增大到一个阀值的时候，region就会等分会两个新的region，之后会有越来越多的region
- Region是Hbase中分布式存储和负载均衡的最小单元，不同Region分布到不同RegionServer上

<center><img src="/assets/images/post/2017-12-20-hbase-construction-theory/hbase-table-region.png" width="500px"/></center>

- Region虽然是分布式存储的最小单元，但并不是存储的最小单元。Region由一个或者多个Store组成，每个store保存一个columns family；每个Strore又由一个memStore和0至多个StoreFile(基于Hfile实现)组成；memStore存储在内存中，StoreFile存储在HDFS上。

<center><img src="/assets/images/post/2017-12-20-hbase-construction-theory/hbase-region-store.png" width="500px"/></center>

### HBASE架构以及基本组件

<center><img src="/assets/images/post/2017-12-20-hbase-construction-theory/hbase-construction.jpg" width="500px"/></center>

- **Client**:包含访问HBase的接口，并维护cache来加快对HBase的访问(比如region的位置信息)；使用HBase RPC机制与HMaster和HRegionServer进行通信，Client与HMaster进行通信进行管理类操作，Client与HRegionServer进行数据读写类操作
- **Master** 
  - 为Region server分配region
  - 负责Region server的负载均衡
  - 发现失效的Region server并重新分配其上的region
  - 管理用户对table的增删改查操作
- **Region Server**
  - Regionserver维护region，处理对这些region的IO请求
  - Regionserver负责切分在运行过程中变得过大的region
- **Zookeeper**
  - 通过选举，保证任何时候，集群中只有一个master，Master与RegionServers启动时会向ZooKeeper注册（避免master单点问题）
  - 存贮所有Region的寻址入口
  - 实时监控Region server的上线和下线信息。并实时通知给Master
  - 存储HBase的schema和table元数据
  - 默认情况下，HBase 管理ZooKeeper 实例，比如， 启动或者停止ZooKeeper

<center><img src="/assets/images/post/2017-12-20-hbase-construction-theory/hbase-master-regionserver.png" width="500px"/></center>

## HBASE写入过程

Client写入 -> 存入MemStore，一直到MemStore满 -> Flush成一个StoreFile，直至增长到一定阈值 -> 触发Compact合并操作 -> 多个StoreFile合并成一个StoreFile，同时进行版本合并和数据删除 -> 当StoreFiles Compact后，逐步形成越来越大的StoreFile -> 单个StoreFile大小超过一定阈值后，触发Split操作，把当前Region Split成2个Region，Region会下线，新Split出的2个孩子Region会被HMaster分配到相应的HRegionServer上，使得原先1个Region的压力得以分流到2个Region上
由此过程可知，HBase只是增加数据，有所得更新和删除操作，都是在Compact阶段做的，所以，用户写操作只需要进入到内存即可立即返回，从而保证I/O高性能


## HBASE容灾(日志恢复)

<center><img src="/assets/images/post/2017-12-20-hbase-construction-theory/hbase-hlog.png" width="500px"/></center>
每次用户执行写入操作，首先是把Log写入到HLog中，HLog是标准的Hadoop Sequence File，由于Log数据量小，而且是顺序写，速度非常快；同时把数据写入到内存MemStore中，成功后返回给Client，所以对Client来说，HBase写的速度非常快，因为数据只要写入到内存中，就算成功了。接着检查MemStore是否已满，如果满了，就把内存中的MemStore Flush到磁盘上，形成一个新的StoreFile。HLog文件定期会滚动出新的，并删除旧的文件（已持久化到StoreFile中的数据）。当HRegionServer意外终止后，HMaster会通过Zookeeper感知到，HMaster首先会处理遗留的 HLog文件，将其中不同Region的Log数据进行拆分，分别放到相应region的目录下，然后再将失效的region重新分配，领取 到这些region的HRegionServer在Load Region的过程中，会发现有历史HLog需要处理，因此会Replay HLog中的数据到MemStore中，然后flush到StoreFiles，完成数据恢复


## HBASE容错性

- **master容错**:Zookeeper重新选择一个新的Master
  - 无Master过程中，数据读取仍照常进行
  - 无master过程中，region切分、负载均衡等无法进行
- **regionServer容错**:定时向Zookeeper汇报心跳，如果一旦时间内未出现心跳，Master将该RegionServer上的Region重新分配到其他RegionServer上，失效服务器上“预写”日志由主服务器进行分割并派送给新的RegionServer
- **zookeeper容错**:Zookeeper是一个可靠地服务，一般配置3或5个Zookeeper实例

## HBASE检索和扫描：如何找到某条记录对应的regionServer

ROOT与META表
Hbase中有两张特殊表，ROOT和META
META：记录了用户表的Region信息，可以有多个region
ROOT：记录了META表的Region信息，ROOT中只有一个region
Zookeeper中记录了ROOT表的location
客户端访问数据的流程： 
Client -> Zookeeper -> -ROOT- -> .META. -> 用户数据表