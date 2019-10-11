---
layout: post
title: "J.U.C之ConcurrentHashMap"
description: "J.U.C之ConcurrentHashMap"
categories: [java并发]
tags: [J.U.C]
redirect_from:
  - /2019/10/10/
---

## 1 ConcurrentHashMap VS HashMap & HashTable
- 多线程环境下，HashMap是非线程安全的，使用HashMap进行put操作会导致链表形成环形数据接口，最终导致死循环，导致CPU利用率达100%。（形成环形数据结构的过程见文章结尾附录）
- HashTab使用synchronized同步锁来保证线程安全，所以在线程竞争激烈的情况下，HashTable的效率非常低下
- 􏲅􏳫􏵆􏳨􏳩􏳬􏳬􏳧􏵆􏳪􏷡􏰩􏷝􏴚􏲉􏰩􏴛􏰜􏳅􏳍􏺛􏰔􏲫􏱜􏱃􏺭􏳛􏺢􏰎􏰏􏴪􏰚􏺹􏲅􏳫􏵆􏳨􏳩􏳬􏳬􏳧􏵆􏳪􏷡􏰩􏷝􏴚􏲉􏰩􏴛􏰜􏳅􏳍􏺛􏰔􏲫􏱜􏱃􏺭􏳛􏺢􏰎􏰏􏴪􏰚􏺹􏲅􏳫􏵆􏳨􏳩􏳬􏳬􏳧􏵆􏳪􏷡􏰩􏷝􏴚􏲉􏰩􏴛􏰜􏳅􏳍􏺛􏰔􏲫􏱜􏱃􏺭􏳛􏺢􏰎􏰏􏴪􏰚􏺹􏲅􏳫􏵆􏳨􏳩􏳬􏳬􏳧􏵆􏳪􏷡􏰩􏷝􏴚􏲉􏰩􏴛􏰜􏳅􏳍􏺛􏰔􏲫􏱜􏱃􏺭􏳛􏺢􏰎􏰏􏴪􏰚􏺹􏲅􏳫􏵆􏳨􏳩􏳬􏳬􏳧􏵆􏳪􏷡􏰩􏷝􏴚􏲉􏰩􏴛􏰜􏳅􏳍􏺛􏰔􏲫􏱜􏱃􏺭􏳛􏺢􏰎􏰏􏴪􏰚􏺹ConcurrentHashMap使用分段锁技术，有效提升了并发访问效率。

## 2 ConcurrentHashMap的结构