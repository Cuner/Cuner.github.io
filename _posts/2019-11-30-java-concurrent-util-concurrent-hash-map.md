---
layout: post
title: "J.U.C之ConcurrentHashMap"
description: "J.U.C之ConcurrentHashMap"
categories: [java并发]
tags: [J.U.C]
redirect_from:
  - /2019/10/10/
---

# 1 ConcurrentHashMap VS HashMap & HashTable
- 多线程环境下，HashMap是非线程安全的，使用HashMap进行put操作会导致链表形成环形数据接口，最终导致死循环，导致CPU利用率达100%。（形成环形数据结构的过程见文章结尾附录）
- HashTab使用synchronized同步锁来保证线程安全，所以在线程竞争激烈的情况下，HashTable的效率非常低下
- 􏲅􏳫􏵆􏳨􏳩􏳬􏳬􏳧􏵆􏳪􏷡􏰩􏷝􏴚􏲉􏰩􏴛􏰜􏳅􏳍􏺛􏰔􏲫􏱜􏱃􏺭􏳛􏺢􏰎􏰏􏴪􏰚􏺹􏲅􏳫􏵆􏳨􏳩􏳬􏳬􏳧􏵆􏳪􏷡􏰩􏷝􏴚􏲉􏰩􏴛􏰜􏳅􏳍􏺛􏰔􏲫􏱜􏱃􏺭􏳛􏺢􏰎􏰏􏴪􏰚􏺹􏲅􏳫􏵆􏳨􏳩􏳬􏳬􏳧􏵆􏳪􏷡􏰩􏷝􏴚􏲉􏰩􏴛􏰜􏳅􏳍􏺛􏰔􏲫􏱜􏱃􏺭􏳛􏺢􏰎􏰏􏴪􏰚􏺹􏲅􏳫􏵆􏳨􏳩􏳬􏳬􏳧􏵆􏳪􏷡􏰩􏷝􏴚􏲉􏰩􏴛􏰜􏳅􏳍􏺛􏰔􏲫􏱜􏱃􏺭􏳛􏺢􏰎􏰏􏴪􏰚􏺹􏲅􏳫􏵆􏳨􏳩􏳬􏳬􏳧􏵆􏳪􏷡􏰩􏷝􏴚􏲉􏰩􏴛􏰜􏳅􏳍􏺛􏰔􏲫􏱜􏱃􏺭􏳛􏺢􏰎􏰏􏴪􏰚􏺹ConcurrentHashMap使用分段锁技术，有效提升了并发访问效率。（容器里面维护了很多锁，每一把锁用于锁容器中的一部分数据，当多线程访问容器中不同数据段的数据时，就不会产生锁竞争，从而有效提高并发访问效率）

# 2 ConcurrentHashMap的结构
链表 NODE
```
static class Node<K,V> implements Map.Entry<K,V> {
	final int hash;
	final K key;
	volatile V val;
	volatile Node<K,V> next;

	Node(int hash, K key, V val, Node<K,V> next) {
		this.hash = hash;
		this.key = key;
		this.val = val;
		this.next = next;
	}

	public final K getKey()       { return key; }
	public final V getValue()     { return val; }
	public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
	public final String toString(){ return key + "=" + val; }
	public final V setValue(V value) {
		throw new UnsupportedOperationException();
	}

	public final boolean equals(Object o) {
		Object k, v, u; Map.Entry<?,?> e;
		return ((o instanceof Map.Entry) &&
				(k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
				(v = e.getValue()) != null &&
				(k == key || k.equals(key)) &&
				(v == (u = val) || v.equals(u)));
	}

	/**
	 * Virtualized support for map.get(); overridden in subclasses.
	 */
	Node<K,V> find(int h, Object k) {
		Node<K,V> e = this;
		if (k != null) {
			do {
				K ek;
				if (e.hash == h &&
						((ek = e.key) == k || (ek != null && k.equals(ek))))
					return e;
			} while ((e = e.next) != null);
		}
		return null;
	}
}
```

其中find()方法，按序遍历链表获取元素



# 附录：多线程场景下hashMap的死循环
