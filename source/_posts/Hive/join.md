---
title: Hive中Join的类型和用法
date: 2019-07-09 12:33:48
tags: 
    - sql
categories: 
    - Hive
---

一图以示之：



![join-all](https://raw.githubusercontent.com/gofreehj/BigData/master/Hive/images/join-all.jpg)

| 类型     | 定义                                                         | 语法             | 等效            |
| -------- | ------------------------------------------------------------ | ---------------- | --------------- |
| 内连接   | 只连接匹配的行                                               | inner join       | join            |
| 左外连接 | 包含左边表的全部行以及右边表中全部匹配的行                   | left outer join  | left join       |
| 右外连接 | 包含右边表的全部行以及左边表中全部匹配的行                   | right outer join | right join      |
| 全外连接 | 包含左、右两个表的全部行，不管在另一边表中是否存在与它们匹配的行 | full out join    | full join       |
| 左半连接 | 以left semi join关键字前面的表为主表，返回主表的KEY也在副表中的记录 | left semi join   |                 |
| 交叉连接 | 生成笛卡尔积—它不适用任何匹配或者选取条件，而是直接讲一个数据源中的每一行与另个数据源的每一行匹配 | cross join       | join 不加on条件 |

### in/exists 

#### exists的执行原理：

> 对外表做loop循环，每次loop循环再对内表（子查询）进行查询，那么因为对内表的查询使用的索引（内表效率高，故可用大表），而外表有多大都需要遍历，不可避免（尽量用小表），故内表大的使用exists，可加快效率；

#### in的执行原理

> 是把外表和内表做hash连接，先查询内表，再把内表结果与外表匹配，对外表使用索引（外表效率高，可用大表），而内表多大都需要查询，不可避免，故外表大的使用in，可加快效率。

#### 使用场景说明：
```sql
SELECT
	c.CustomerId,
	c.CompanyName
FROM
	Customers c
WHERE
	EXISTS
	(
		SELECT OrderID FROM Orders o WHERE o.CustomerID = c.CustomerID
	)
```

> 分析：这里使用exists的原因是，订单表里面可能记录很大，而客户表是一个相当较小的表，这样查询的话 是一种优化方式。

```sql
SELECT * FROM Orders WHERE CustomerId in (id1,id2,id3);

SELECT
	*
FROM
	Orders
WHERE
	CustomerID in 
	(
		SELECT CustomerId FROM Customers WHERE customer_type = 1
	)	
```

> 分析 ：这里我只查找客户编号是id1,id2,id3 的人的订单信息.  in就特别合适了。

**注意：in后面子查询可以使用任何子查询（或常数），exists后面的只能是相关子查询（不然没意义）**



### left semi join 与 inner join 相同点与区别

hive 的 join 类型有好几种，其实都是把 MR 中的几种方式都封装实现了，其中 join on、left semi join 算是里边具有代表性，且使用频率较高的 join 方式。

1.联系

他们都是 hive join 方式的一种，join on 属于 common join（shuffle join/reduce join），而 left semi join 则属于 map join（broadcast join）的一种变体，从名字可以看出他们的实现原理有差异。

2.区别

（1）Semi Join，也叫半连接，是从分布式数据库中借鉴过来的方法。它的产生动机是：对于reduce side join，跨机器的数据传输量非常大，这成了join操作的一个瓶颈，如果能够在map端过滤掉不会参加join操作的数据，则可以大大节省网络IO，提升执行效率。
实现方法很简单：选取一个小表，假设是File1，将其参与join的key抽取出来，保存到文件File3中，File3文件一般很小，可以放到内存中。在map阶段，使用DistributedCache将File3复制到各个TaskTracker上，然后将File2中不在File3中的key对应的记录过滤掉，剩下的reduce阶段的工作与reduce side join相同。
由于 hive 中没有 in/exist 这样的子句（新版将支持），所以需要将这种类型的子句转成 left semi join。left semi join 是只传递表的 join key 给 map 阶段 , 如果 key 足够小还是执行 map join, 如果不是则还是 common join。

（2）left semi join 子句中右边的表只能在 ON 子句中设置过滤条件，在 WHERE 子句、SELECT 子句或其他地方过滤都不行。

（3）对待右表中重复key的处理方式差异：因为 left semi join 是 in(keySet) 的关系，遇到右表重复记录，左表会跳过，而 join on 则会一直遍历。

最后的结果是这会造成性能，以及 join 结果上的差异。

（4）left semi join 中最后 select 的结果只许出现左表，因为右表只有 join key 参与关联计算了，而 join on 默认是整个关系模型都参与计算了。

> 注意：大多数情况下 JOIN ON 和 left semi on 是对等的，但是在上述情况下会出现重复记录，导致结果差异，所以大家在使用的时候最好能了解这两种方式的原理，避免掉“坑”。



示例：

```sql
SELECT a.key, a.val FROM a WHERE a.key in (SELECT b.key FROM b);
```

可以被改写为：

 ```sql
SELECT a.key, a.val FROM a LEFT SEMI JOIN b on (a.key = b.key)
 ```

特点：

1、left semi join 的限制是， JOIN 子句中右边的表只能在 ON 子句中设置过滤条件，在 WHERE 子句、SELECT 子句或其他地方过滤都不行。

2、left semi join 是只传递表的 join key 给 map 阶段，因此left semi join 中最后 select 的结果只许出现左表。

3、因为 left semi join 是 in(keySet) 的关系，遇到右表重复记录，左表会跳过，而 join 则会一直遍历。这就导致右表有重复值得情况下 left semi join 只产生一条，join 会产生多条，也会导致 left semi join 的性能更高。 

比如以下A表和B表进行 join 或 left semi join，然后 select 出所有字段，结果区别如下：

![left_smi_join](https://raw.githubusercontent.com/gofreehj/BigData/master/Hive/images/left_smi_join.jpg)

注意：蓝色叉的那一列实际是不存在left semi join中的，因为最后 select 的结果只许出现左表。

 

---

参考：

[Hive 中的 LEFT SEMI JOIN 与 JOIN ON](https://www.cnblogs.com/wqbin/p/11023008.html)

[Hive中Join的类型和用法](https://www.cnblogs.com/liupengpengg/p/7908274.html)

https://zhuanlan.zhihu.com/p/25435517

[hive 的 left semi join 讲解](https://blog.csdn.net/happyrocking/article/details/79885071)

[MapJoin和ReduceJoin区别及优化](https://blog.csdn.net/qq_17776287/article/details/78567514)