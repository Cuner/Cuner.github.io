---
layout: post
title: "paxos与raft一致性协议"
description: "paxos与raft一致性协议"
categories: [一致性协议]
tags: [一致性协议]
redirect_from:
  - /2018/08/04/
---
# 介绍
下面是关于两种一致性协议的相关论文的详细讲解，这里就不再赘述
- PAXOS
    - [《Paxos Made Simple》论文翻译](https://www.jianshu.com/p/6d01a8d2df9f)
    - [《Paxos Made Simple》原文](/assets/pdf/paxos-simple1.pdf)
- RAFT
    - [《In search of an Understandable Consensus Algorithm (Extended Version)》中文翻译](http://www.infoq.com/cn/articles/raft-paper)
    - [《In search of an Understandable Consensus Algorithm (Extended Version)》原文](/assets/pdf/raft.pdf)
    - [Raft Understandable Distributed Consensus](http://thesecretlivesofdata.com/raft/)
    - [raft implementation by Go](https://github.com/coreos/etcd/tree/master/raft#usage)
    
# 异同

