---
layout: post
title: "数据库锁机制"
description: "数据库锁机制"
categories: [database]
tags: [mysql]
redirect_from:
  - /2017/03/29/
---
# 数据库锁机制

## 为什么需要锁

- 死锁问题
- 并发问题导致的不正确数据的读取和存储，破坏数据一致性
 1. 丢失更新：当两个或多个事务选择同一行，然后基于最初选定的值更新该行时，由于每个事务都不知道其他事务的存在，就会发生丢失更新问题－－最后的更新覆盖了由其他事务所做的更新。例如，两个编辑人员制作了同一文档的电子副本。每个编辑人员独立地更改其副本，然后保存更改后的副本，这样就覆盖了原始文档。最后保存其更改副本的编辑人员覆盖另一个编辑人员所做的更改。如果在一个编辑人员完成并提交事务之前，另一个编辑人员不能访问同一文件，则可避免此问题
 2. 脏读：一个事务正在对一条记录做修改，在这个事务完成并提交前，这条记录的数据就处于不一致状态；这时，另一个事务也来读取同一条记录，如果不加控制，第二个事务读取了这些“脏”数据，并据此做进一步的处理，就会产生未提交的数据依赖关系。这种现象被形象地叫做"脏读"。
 3. 不可重复读：一个事务在读取某些数据后的某个时间，再次读取以前读过的数据，却发现其读出的数据已经发生了改变、或某些记录已经被删除了！这种现象就叫做“不可重复读”。
 4. 幻读：一个事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据，这种现象就称为“幻读”。

## 锁的分类

### 数据库维度

- 共享锁:  
用于不更改或不更新数据的操作（只读操作）。共享锁允许并发事务读取同一个资源，数据资源上存在共享锁时，任何其他事务不允许修改数据
- 排它锁:  
用于数据修改，确保不会同时多重更新同一数据。资源上存在排他锁时，其他任何事务不允许给资源上锁，当资源上有其他锁时，也无法对其加上排它锁

PS：只有共享锁与共享锁相互兼容，共享锁与排它锁、排它锁之间都互不兼容

- 意向锁

- 更新锁   
 1. 用于可更新的资源中，防止多个会话在读取、锁定以及随后可能进行的资源更新时发生的死锁问题。
 2. 通常形式的死锁：一般一个更新模式由一个事务组成，此事务读取记录，获取资源的共享锁，然后修改行，此操作要求锁转换为排它锁。如果两个事务同时获得了资源上的共享锁，然后试图同时更新数据，则一个事务尝试将锁转换为排它锁。由于一个事务的排它锁与另一事务的共享锁的不兼容，从共享锁到排它锁的转换必须要等待一段时间，发送锁等待。而第二个事务同时也试图获取排它锁进行更新。由于两个事务都要转化为排它锁，并且每个事务都要等待另一个事务释放掉共享锁，因此发生死锁
 3. 更新锁一次只有一个事务可以获取资源的更新锁。如果事务修改资源，那么更新锁转化为排它锁，否则转化为共享锁。当资源上存在更新锁时，允许资源被读取（即更新锁与共享锁兼容），但不允许资源被修改
 4. 一般来说，在执行UPDATE操作时，SQL SERVER会使用到更新锁而不是依次加上共享锁和排它锁，已经回避了这种通常形式的死锁，更新锁与意向锁相互兼容

### 锁机制

DBMS | SELECT | UPDATE | INSERT | DELETE 
----|------|----|---|---
MySQL(InnoDB) | 不加锁 | 排它锁 | 排它锁 | 排它锁 
SQL SERVER | 共享锁 | 更新锁 | 排它锁 | 排它锁

 

这两种锁机制的区别在于MySQL的查询与更新操作互相不阻塞；而SQL SERVER的更新锁转化成排它锁之前，其查询与更新操作互相不阻塞，当更新锁要转化为排它锁时，需要等待共享锁的释放，当更新锁转化为排它锁后，查询数据需要等待排它锁的释放。

参考：  
[数据库锁机制](http://blog.csdn.net/samjustin1/article/details/52210125)  
[InnoDB锁机制](http://blog.chinaunix.net/uid-24111901-id-2627857.html)  
[SQL SERVER锁机制](http://blog.itpub.net/13651903/viewspace-1091664/)


### 程序员思想维度
- 悲观锁  
 1. 悲观并发控制（又名“悲观锁”，Pessimistic Concurrency Control，缩写“PCC”）是一种并发控制的方法。它可以阻止一个事务以影响其他用户的方式来修改数据。如果一个事务执行的操作都某行数据应用了锁，那只有当这个事务把锁释放，其他事务才能够执行与该锁冲突的操作。悲观锁的实现，往往依靠数据库提供的锁机制。悲观并发控制主要用于数据争用激烈的环境，以及发生并发冲突时使用锁保护数据的成本要低于回滚事务的成本的环境中。
 2. 悲观并发控制实际上是“先取锁再访问”的保守策略，为数据处理的安全提供了保证。但是在效率方面，处理加锁的机制会让数据库产生额外的开销，还有增加产生死锁的机会；另外，在只读型事务处理中由于不会产生冲突，也没必要使用锁，这样做只能增加系统负载；还有会降低了并行性，一个事务如果锁定了某行数据，其他事务就必须等待该事务处理完才可以处理那行数

- 乐观锁  
 1. 乐观并发控制（又名“乐观锁”，Optimistic Concurrency Control，缩写“OCC”）是一种并发控制的方法。它假设多用户并发的事务在处理时不会彼此互相影响，各事务能够在不产生锁的情况下处理各自影响的那部分数据。在提交数据更新之前，每个事务会先检查在该事务读取数据后，有没有其他事务又修改了该数据。如果其他事务有更新的话，正在提交的事务会进行回滚。
 2. 乐观并发控制相信事务之间的数据竞争(data race)的概率是比较小的，因此尽可能直接做下去，直到提交的时候才去锁定，所以不会主动产生任何锁和死锁。但是在并发量高的情况下，可能导致某次数据修改多次重试，影响单次成功操作的时间。
 3. 数据版本实现乐观锁：实现数据版本有两种方式，第一种是使用版本号，第二种是使用时间戳。使用版本号时，可以在数据初始化时指定一个版本号，每次对数据的更新操作都对版本号执行+1操作。并判断当前版本号是不是该数据的最新的版本号。
```
update table 
set date=1,version=version+1
where id=#{id} and version=#{version};
```

参考：  
[乐观锁与悲观锁](http://www.open-open.com/lib/view/open1452046967245.html)

## 乐观锁另一种实现方式CAS

CAS是项乐观锁技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。

CAS 操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。无论哪种情况，它都会在 CAS 指令之前返回该位置的值。CAS 有效地说明了“我认为位置 V 应该包含值 A；如果包含该值，则将 B 放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可。”这其实和乐观锁的冲突检查+数据更新的原理是一样的。

java.util.concurrent(J.U.C)就是建立在CAS之上的。相对于对于synchronized这种阻塞算法，CAS是非阻塞算法的一种常见实现。所以J.U.C在性能上有了很大的提升。
```
public class AtomicInteger extends Number implements java.io.Serializable {
    private volatile int value;  
 
    public final int get() {  
        return value;  
    }  
 
    public final int getAndIncrement() {  
        for (;;) {  
            int current = get();  
            int next = current + 1;  
            if (compareAndSet(current, next))  
                return current;  
        }  
    }  
 
    public final boolean compareAndSet(int expect, int update) {  
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);  
    }  
}
```

参考：  
[乐观锁的一种实现方式—CAS](http://www.importnew.com/20472.html)

## 案例1：初审统计数据迁移

- 迁移背景  
原有的统计方式采用的是实时count的方法获取统计数据，造成的问题是查询慢且无法获取长时间段的统计数据（sql超时）、无法获取某日统计数据的快照（前一天的待审核数据会变成今天的审核通过数据）
- 采用迁移方式  
使用raptor迁移平台，扫描审核记录表，取出累计统计数据后进行加1操作，然后更新到统计表中。由于平台特性，数据迁移过程具有高并发性，由于强行采用先读取后更新的方式，会造成丢失更新的情况，于是这里考虑采用CAS  

step 1:
```
select id,passCount,rejectCount,hideCount,warnCount,waitCount 
from book.TradeItemAuditCount 
where type = #{type} and date = #{date} and editor = #{editor} 
and isDeleted = 0 limit 1

```
step 2:【失败重试】
```
update book.TradeItemAuditCount 
set passCount = #{passCount} , rejectCount = #{rejectCount} , hideCount = #{hideCount} , warnCount = #{warnCount} , waitCount = #{waitCount} , updated = #{updated}
where id = #{id} and passCount = #{oldPassCount} and rejectCount = #{oldRejectCount} and hideCount = #{oldHideCount} and warnCount = #{oldWarnCount} and waitCount = #{oldWaitCount}
and isDeleted = 0 limit 1
```

- 处理结果  
一共扫描审核结果4335668条数据，对重试次数超过100的更新操作进行记录，发现更新操作出现大部分的重试，任务本身DB写操作的qps较低【都不需要通过控制台限制速率..】

## 案例2：商品库存
- 商品库存与上述案例1一致，都是对数据记录进行加减操作，发现库存的更新方式如下：
```
update 库存表
set stock=stock-1
where id=#{id}
```

- 直接使用数据库的排它锁就简单的避免了并发导致的丢失更新问题，之前提到的一次只有一个事务拥有资源的排它锁，并发的更新操作都试图占有资源的排它锁，当资源上存在排它锁时，其他更新操作需要等待锁的释放

- 相比案例1的解决方案，案例2的解决方式直接使用了MySQL InnoDB更新操作本身就拥有的排它锁，不需要额外的开销，而案例1不必要的查询操作以及多次的重试操作严重影响到了数据迁移的性能，所以案例1是反面例子..

## 案例3：商品打标
- 随着上打标的qps上涨，出现达标更新数据丢失的情况
```
{"tags":"16,32,233,22","itemState":1,"hd":"ai:4|nd:18","au":"baoming"}
```

- 处理方案
 1. 乐观锁：采用CAS  
 ```
update TradeItem
set extra=#{extra}
where tradeItemId=#{tradeItemId} and extra=#{oldExtra}
 ```
 这里使用长字符串做更新条件，会影响到SQL性能？？？
 2. 乐观锁：采用数据版本 
 表中新增version字段标识数据版本，作为数据更新的检查方式
```
update TradeItem
set extra=#{extra} , version=version+1
where tradeItemId=#{tradeItemId} and version=#{version}
```
此方案改造较大，还需要为表新增字段，而且采用乐观锁拥有这一律的弊端：重试带来的时间代价，一旦并发量上涨，某次更新操作的重试次数也会随之上涨，直接影响到暴露服务的响应时间。【限制重试次数能够一定程度上控制更新操作的响应时间，但是仍然会出现更新丢失的现象（让调用方进行重试操作，分摊单次请求的响应时间？）】
 3. 悲观锁  
更新丢失的根本原因是执行查询、修改两个操作之间数据被另一事务修改了，单纯的UPDATE操作其实也是进行着先查询后修改的操作，没有产生更新丢失是因为数据上存在排它锁（sql server则是更新锁），在执行期间并不允许其他修改。同理我们将要打标的商品记录加上排它锁或者更新锁就能解决问题。  
MySQL:
```
start transaction;
SELECT extra
FROM TradeItem 
WHERE tradeItemId=#{tradeItemId}
FOR UPDATE;
UPDATE TradeItem 
SET extra = bdo.AddTag(tag,extra)
WHERE tradeItemId=#{tradeItemId};
commit;
```
SQL SERVER:
```
BEGIN TRANSACTION --开始一个事务
SELECT extra
FROM TradeItem WITH (UPDLOCK)
WHERE tradeItemId=#{tradeItemId}
UPDATE TradeItem 
SET extra = bdo.AddTag(tag,extra)
WHERE tradeItemId=#{tradeItemId}
COMMIT TRANSACTION --提交事务
```
该方案避免了重试带来的开销，同时使用排它锁（更新锁）也没有额外增加锁的开销【直接把commit写到sql里面会被raptor拦截，只能使用dateSource.getConnection设置自动提交为false后提交】

## 悲观锁乐观锁的取舍