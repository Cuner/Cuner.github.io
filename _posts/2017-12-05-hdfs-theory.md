---
layout: post
title: "HDFS分布式文件系统"
description: "HDFS分布式文件系统"
categories: [database]
tags: [hdfs]
redirect_from:
  - /2017/12/5/
---
# HDFS设计理念

Hadoop分布式文件系统(HDFS)被设计成适合运行在通用硬件上的分布式文件系统，HDFS是一个高度容错性的系统，适合部署在廉价的机器上。HDFS能提供高吞吐量的数据访问，非常适合大规模数据集上的应用。

- <b>存储超大文件:</b>运行在HDFS上的应用具有很大的数据集。HDFS上的一个典型文件大小一般都在G字节至T字节。因此，HDFS被调节以支持大文件存储。它能提供整体上高的数据传输带宽，能在一个集群里扩展到数百个节点。一个单一的HDFS实例应该能支撑数以千万计的文件。
- <b>一次写入、多次读取:</b>HDFS存储的数据集作为hadoop的分析对象。在数据集生成后，长时间在此数据集上进行各种分析。每次分析都将设计该数据集的大部分数据甚至全部数据，比之数据访问的低延迟问题，更关键的在于数据访问的高吞吐量，因此读取整个数据集的时间延迟比读取第一条记录的时间延迟更重要。
- <b>运行在普通廉价的服务器上:</b>HDFS设计理念之一就是让它能运行在普通的硬件之上，即便硬件出现故障，也可以通过容错策略(错误检测和快速、自动的恢复)来保证数据的高可用。

# HDFS基本概念
HDFS暴露了文件系统的名字空间，用户能够以文件的形式在上面存储数据。从内部看，一个文件其实被分成一个或多个数据块，这些块存储在一组Datanode上。Namenode执行文件系统的名字空间操作，比如打开、关闭、重命名文件或目录。它也负责确定数据块到具体Datanode节点的映射。Datanode负责处理文件系统客户端的读写请求。在Namenode的统一调度下进行数据块的创建、删除和复制
- <b>数据块(block):</b>大文件会被分割成多个block进行存储，block大小默认为64MB。每一个block会在多个datanode上存储多份副本，默认是3份。
- <b>namenode:</b>namenode是一个中心服务器，负责管理文件目录、文件和block的对应关系以及block和datanode的对应关系。
- <b>datanode:</b>datanode就负责存储了，当然大部分容错机制都是在datanode上实现的。

# HDFS基本架构图
<center><img src="/assets/images/post/2017-12-05-hdfs-theory/hdfs-construction.png" width="500px"/></center>
Rack 是指机架的意思，一个block的三个副本通常会保存到两个或者两个以上的机柜中（当然是机柜中的服务器），这样做的目的是做防灾容错。  

每个Datanode节点周期性地向Namenode发送心跳信号。网络割裂可能导致一部分Datanode跟Namenode失去联系。Namenode通过心跳信号的缺失来检测这一情况，并将这些近期不再发送心跳信号Datanode标记为宕机，不会再将新的IO请求发给它们。任何存储在宕机Datanode上的数据将不再有效。Datanode的宕机可能会引起一些数据块的副本系数低于指定值，Namenode不断地检测这些需要复制的数据块，一旦发现就启动复制操作。在下列情况下，可能需要重新复制：某个Datanode节点失效，某个副本遭到损坏，Datanode上的硬盘错误，或者文件的副本系数增大。

# HDFS写文件流程
<center><img src="/assets/images/post/2017-12-05-hdfs-theory/hdfs-write.png" width="500px"/></center>

当客户端向HDFS文件写入数据的时候，一开始是写到本地临时文件中。假设该文件的副本系数设置为3，当本地临时文件累积到一个数据块的大小时，客户端会从Namenode获取一个Datanode列表用于存放副本。然后客户端开始向第一个Datanode传输数据，第一个Datanode一小部分一小部分(4 KB)地接收数据，将每一部分写入本地仓库，并同时传输该部分到列表中第二个Datanode节点。第二个Datanode也是这样，一小部分一小部分地接收数据，写入本地仓库，并同时传给第三个Datanode。最后，第三个Datanode接收数据并存储在本地。因此，Datanode能流水线式地从前一个节点接收数据，并在同时转发给下一个节点，数据以流水线的方式从前一个Datanode复制到下一个。
【ps】另一种数据写入的方式是采用多路写，也就是同时别写入到三个Datanode里面。管道写需要时间去完成，所以它有很高的延迟，但是它能更好地利用网络带宽；多路写有着比较低的延迟，因为客户端只需要等待最慢的DataNode确认（假设其余都已成功确认）。但是写入需要共享发送服务器的网络带宽，这对于有着很高负载的系统来说是一个瓶颈。

# HDFS读文件流程
<center><img src="/assets/images/post/2017-12-05-hdfs-theory/hdfs-read.png" width="500px"/></center>
