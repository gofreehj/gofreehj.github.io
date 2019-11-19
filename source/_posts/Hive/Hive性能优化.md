---
title: Hive性能优化上的一些总结
date: 2019-07-07 12:33:48
tags: 
    - sql
categories: 
    - Hive
---


> https://blog.csdn.net/mrlevo520/article/details/76339075



Hive性能优化上的一些总结
https://www.cnblogs.com/smartloli/p/4356660.html
https://www.cnblogs.com/frankdeng/tag/Hive/
https://blog.csdn.net/mrlevo520/article/details/76339075



# Hive 性能优化上的一些总结

**_注意，本文百分之九十来源于此文:[Hive 性能优化](http://www.cnblogs.com/smartloli/p/4356660.html)，很感谢作者的细心整理，其中有些部分我做了补充和追加，要是有什么写的不对的地方，请留言赐教，谢谢_**

# 前言

> 今天电话面试突然被涉及到 hive 上有没有做过什么优化，当时刚睡醒，迷迷糊糊的没把以前实习的中遇到的一些问题阐述清楚，这里顺便转载一篇并来做一下总结

* * *

# 介绍

> 首先，我们来看看 Hadoop 的计算框架特性，在此特性下会衍生哪些问题？

* 数据量大不是问题，数据倾斜是个问题。

* jobs 数比较多的作业运行效率相对比较低，比如即使有几百行的表，如果多次关联多次汇总，产生十几个 jobs，耗时很长。原因是 map reduce 作业初始化的时间是比较长的。

* sum,count,max,min 等 UDAF，不怕数据倾斜问题, hadoop 在 map 端的汇总合并优化，使数据倾斜不成问题。

*   count(distinct), 在数据量大的情况下，效率较低，如果是多 count(distinct ) 效率更低，因为 count(distinct) 是按 group by 字段分组，按 distinct 字段排序，一般这种分布方式是很倾斜的。举个例子：比如男 uv, 女 uv，像淘宝一天 30 亿的 pv，如果按性别分组，分配 2 个 reduce, 每个 reduce 处理 15 亿数据。

    　　面对这些问题，我们能有哪些有效的优化手段呢？下面列出一些在工作有效可行的优化手段：

*   好的模型设计事半功倍。

* 解决数据倾斜问题。

* 减少 job 数。

* 设置合理的 map reduce 的 task 数，能有效提升性能。(比如，10w + 级别的计算，用 160 个 reduce，那是相当的浪费，1 个足够)。

* 了解数据分布，自己动手解决数据倾斜问题是个不错的选择。set hive.groupby.skewindata=true; 这是通用的算法优化，但算法优化有时不能适应特定业务背景，开发人员了解业务，了解数据，可以通过业务逻辑精确有效的解决数据倾斜问题。

* 数据量较大的情况下，慎用 count(distinct)，count(distinct) 容易产生倾斜问题。

* 对小文件进行合并，是行至有效的提高调度效率的方法，假如所有的作业设置合理的文件数，对云梯的整体调度效率也会产生积极的正向影响。

* 优化时把握整体，单个作业最优不如整体最优。

     而接下来，我们心中应该会有一些疑问，影响性能的根源是什么？

     

# 性能低下的根源

> hive 性能优化时，把 HiveQL 当做 M/R 程序来读，即从 M/R 的运行角度来考虑优化性能，从更底层思考如何优化运算性能，而不仅仅局限于逻辑代码的替换层面。
> 
>  RAC（Real Application Cluster）真正应用集群就像一辆机动灵活的小货车，响应快；Hadoop 就像吞吐量巨大的轮船，启动开销大，如果每次只做小数量的输入输出，利用率将会很低。所以用好 Hadoop 的首要任务是增大每次任务所搭载的数据量。

　　**Hadoop 的核心能力是 parition 和 sort，因而这也是优化的根本。**

　　_观察 Hadoop 处理数据的过程，有几个显著的特征_：

*   数据的大规模并不是负载重点，造成运行压力过大是因为运行数据的倾斜。
*   jobs 数比较多的作业运行效率相对比较低，比如即使有几百行的表，如果多次关联对此汇总，产生几十个 jobs，将会需要 30 分钟以上的时间且大部分时间被用于作业分配，初始化和数据输出。M/R 作业初始化的时间是比较耗时间资源的一个部分。
*   在使用 SUM，COUNT，MAX，MIN 等 UDAF 函数时，**不怕数据倾斜问题**，Hadoop 在 Map 端的汇总合并优化过，使数据倾斜不成问题。
*   COUNT(DISTINCT) 在数据量大的情况下，效率较低，如果多 COUNT(DISTINCT) 效率更低，因为 COUNT(DISTINCT) 是按 GROUP BY 字段分组，按 DISTINCT 字段排序，一般这种分布式方式是很倾斜的；比如：男 UV，女 UV，淘宝一天 30 亿的 PV，如果按性别分组，分配 2 个 reduce, 每个 reduce 处理 15 亿数据。
* 数据倾斜是导致效率大幅降低的主要原因，可以采用多一次 Map/Reduce 的方法， 避免倾斜。

     **最后得出的结论是：避实就虚，用 job 数的增加，输入量的增加，占用更多存储空间，充分利用空闲 CPU 等各种方法，分解数据倾斜造成的负担。**

     


# 优化性能

## 配置角度优化

### map 阶段优化

> Map 阶段的优化，主要是确定合适的 map 数。那么首先要了解 map 数的计算公式, 另外要说明的是，这个优化只是针对 Hive 0.9 版本。

```sql
num_map_tasks =max[${mapred.min.split.size},min(${dfs.block.size},${mapred.max.split.size})]
```

*   mapred.min.split.size: 指的是数据的最小分割单元大小；min 的默认值是 1B
*   mapred.max.split.size: 指的是数据的最大分割单元大小；max 的默认值是 256MB
*   dfs.block.size: 指的是 HDFS 设置的数据块大小。个已经指定好的值，而且这个参数默认情况下 hive 是识别不到的

> 通过调整 max 可以起到调整 map 数的作用，减小 max 可以增加 map 数，增大 max 可以减少 map 数。需要提醒的是，**直接调整 mapred.map.tasks 这个参数是没有效果的**。

### reduce 阶段优化

> 这里说的 reduce 阶段，是指前面流程图中的 reduce phase（实际的 reduce 计算）而非图中整个 reduce task。Reduce 阶段优化的主要工作也是选择合适的 reduce task 数量, 与 map 优化不同的是，reduce 优化时，**可以直接设置 mapred.reduce.tasks 参数从而直接指定 reduce 的个数**

```
num_reduce_tasks = min[${hive.exec.reducers.max}(${input.size}/${hive.exec.reducers.bytes.per.reducer})]
```

*   hive.exec.reducers.max：此参数从 Hive 0.2.0 开始引入。在 Hive 0.14.0 版本之前默认值是 999；而从 Hive 0.14.0 开始，默认值变成了 1009，这个参数的含义是最多启动的 Reduce 个数

*   hive.exec.reducers.bytes.per.reducer：此参数从 [Hive](https://www.iteblog.com/archives/tag/hive/) 0.2.0 开始引入。在 Hive 0.14.0 版本之前默认值是 1G(1,000,000,000)；而从 Hive 0.14.0 开始，默认值变成了 256M(256,000,000)，可以参见 HIVE-7158 和 HIVE-7917。这个参数的含义是每个 Reduce 处理的字节数。比如输入文件的大小是 1GB，那么会启动 4 个 Reduce 来处理数据。

> 也就是说，根据输入的数据量大小来决定 Reduce 的个数，默认 Hive.exec.Reducers.bytes.per.Reducer 为 1G，而且 Reduce 个数不能超过一个上限参数值，这个参数的默认取值为 999。所以我们可以调整 Hive.exec.Reducers.bytes.per.Reducer 来设置 Reduce 个数。

**需要注意的是：**

1.  Reduce 的个数对整个作业的运行性能有很大影响。如果 Reduce 设置的过大，**那么将会产生很多小文件，对 NameNode 会产生一定的影响**，而且整个作业的运行时间未必会减少；如果 Reduce 设置的过小，那么单个 Reduce 处理的数据将会加大，**很可能会引起 OOM 异常**。
2.  如果设置了`mapred.reduce.tasks/mapreduce.job.reduces`参数，那么 **Hive 会直接使用它的值作为 Reduce 的个数**；
3.  如果`mapred.reduce.tasks/mapreduce.job.reduces`的值没有设置（也就是 - 1），那么 Hive 会根据输入文件的大小估算出 Reduce 的个数。根据输入文件估算 Reduce 的个数可能未必很准确，因为 Reduce 的输入是 Map 的输出，而 Map 的输出可能会比输入要小，所以最准确的数根据 Map 的输出估算 Reduce 的个数。

### 列裁剪

> Hive 在读数据的时候，可以只读取查询中所需要用到的列，而忽略其它列。 例如，若有以下查询：

```
SELECT a,b FROM q WHERE e<10;
```

> 在实施此项查询中，Q 表有 5 列（a，b，c，d，e），Hive 只读取查询逻辑中真实需要 的 3 列 a、b、e，而忽略列 c，d；这样做节省了读取开销，中间表存储开销和数据整合开销。
> 
>  裁剪所对应的参数项为：hive.optimize.cp=true（默认值为真）

补充：在我实习的操作过程中，也有用到这个道理，也就是多次 join 的时候，考虑到只需要的指标，而不是为了省事使用 select * 作为子查询

### 分区裁剪

> 可以在查询的过程中减少不必要的分区。 例如，若有以下查询：

```
SELECT 
* 
FROM 
(   
SELECTT 
    a1,
    COUNT(1) 
FROM T 
GROUP BY a1
)subq    # 建议贴边写，这样容易检查是否是中文括号！
WHERE subq.prtn=100; #（多余分区）

SELECT 
* 
FROM 
T1 
JOIN 
(
  SELECT 
  * 
  FROM T2
)subq 
ON (T1.a1=subq.a2) 
WHERE subq.prtn=100;
```

> 查询语句若将 “subq.prtn=100” 条件放入子查询中更为高效，可以减少读入的分区 数目。 Hive 自动执行这种裁剪优化。
> 
>  分区参数为：hive.optimize.pruner=true（默认值为真）

补充：实际集群操作过程中，加分区是重中之重，不加分区的后果非常可能把整个队列资源占满，而导致 io 读写异常，无法登陆服务器及 hive！切记切记分区操作和 limit 操作

### JOIN 操作

> 在编写带有 join 操作的代码语句时，应该将条目少的表 / 子查询放在 Join 操作符的左边。 因为在 Reduce 阶段，位于 Join 操作符左边的表的内容会被加载进内存，载入条目较少的表 可以有效减少 OOM（out of memory）即内存溢出。所以对于同一个 key 来说，对应的 value 值小的放前，大的放后，这便是 “小表放前” 原则。 若一条语句中有多个 Join，依据 Join 的条件相同与否，有不同的处理方法。

#### JOIN 原则

> 在使用写有 Join 操作的查询语句时有一条原则：应该将条目少的表 / 子查询放在 Join 操作符的左边。原因是在 Join 操作的 Reduce 阶段，位于 Join 操作符左边的表的内容会被加载进内存，将条目少的表放在左边，可以有效减少发生 OOM 错误的几率。对于一条语句中有多个 Join 的情况，如果 Join 的条件相同，**一句话就是小表在左边**比如查询：

```sql
INSERT OVERWRITE TABLE pv_users 
SELECT 
    pv.pageid,
    u.age 
FROM page_view p 
JOIN user u ON (pv.userid = u.userid) 
JOIN newuser x ON (u.userid = x.userid);  
```

*   如果 Join 的 key 相同，不管有多少个表，都会则会合并为一个 Map-Reduce
*   一个 Map-Reduce 任务，而不是 ‘n’ 个
*   在做 OUTER JOIN 的时候也是一样

> 如果 Join 的条件不相同，比如：

```sql
INSERT OVERWRITE TABLE pv_users 
SELECT 
   pv.pageid, 
   u.age 
FROM page_view p 
JOIN user u ON (pv.userid = u.userid) 
JOIN newuser x on (u.age = x.age);   
```

> Map-Reduce 的任务数目和 Join 操作的数目是对应的，上述查询和以下查询是等价的：

```sql
INSERT OVERWRITE TABLE tmptable 
SELECT 
* 
FROM page_view p 
JOIN 
user u 
ON (pv.userid = u.userid);

INSERT OVERWRITE TABLE pv_users 
SELECT 
    x.pageid, 
    x.age 
FROM tmptable x 
JOIN 
newuser y 
ON (x.age = y.age);    
```

#### MAP JOIN 操作

> **如果你有一张表非常非常小，而另一张关联的表非常非常大的时候，你可以使用 mapjoin** 此 Join 操作在 Map 阶段完成，不再需要 Reduce，**也就不需要经过** [Shuffle 过程](http://langyu.iteye.com/blog/992916)**，从而能在一定程度上节省资源提高 JOIN 效率**前提条件是需要的数据在 Map 的过程中可以访问到。比如查询：

```sql
INSERT OVERWRITE TABLE pv_users 
   SELECT /*+ MAPJOIN(pv) */ pv.pageid, u.age 
   FROM page_view pv 
     JOIN user u ON (pv.userid = u.userid);    
```

　　可以在 Map 阶段完成 Join，如图所示：

![](https://img-blog.csdn.net/20170728205224816?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTXJMZXZvNTIw/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

　　相关的参数为：

*   **hive.join.emit.interval = 1000**
*   **hive.mapjoin.size.key = 10000**
*   **hive.mapjoin.cache.numrows = 10000**

参考于 [Hive MapJoin](http://www.cnblogs.com/MOBIN/p/5702580.html)：值得注意的是，Hive 版本 0.11 之后，Hive 默认启动该优化，也就是不在需要显示的使用 MAPJOIN 标记，其会在必要的时候触发该优化操作将普通 JOIN 转换成 MapJoin

两个属性来设置该优化的触发时机

```sql
hive.auto.convert.join
```

默认值为 true，自动开户 MAPJOIN 优化

```sql
hive.mapjoin.smalltable.filesize
```

默认值为 2500000(25M), 通过配置该属性来确定使用该优化的表的大小，如果表的大小小于此值就会被加载进内存中

### GROUP BY 操作

> 进行 GROUP BY 操作时需要注意一下几点：

*   Map 端部分聚合

    　　事实上并不是所有的聚合操作都需要在 reduce 部分进行，很多聚合操作都可以先在 Map 端进行部分聚合，然后 reduce 端得出最终结果。

        　　这里需要修改的参数为：

`hive.map.aggr=true（用于设定是否在 map 端进行聚合，默认值为真）` `hive.groupby.mapaggr.checkinterval=100000（用于设定 map 端进行聚合操作的条目数）`

*   有数据倾斜时进行负载均衡

    　　此处需要设定 **hive.groupby.skewindata**，当选项设定为 true 是，生成的查询计划有两个 MapReduce 任务。

    1.  在第一个 MapReduce 中，map 的输出结果集合会随机分布到 reduce 中， 每个 reduce 做部分聚合操作，并输出结果。这样处理的结果是，**相同的 Group By Key 有可能分发到不同的 reduce 中，从而达到负载均衡的目的**；
    2.  第二个 MapReduce 任务再根据预处 理的数据结果按照 Group By Key 分布到 reduce 中（这个过程可以保证相同的 Group By Key 分布到同一个 reduce 中），最后完成最终的聚合操作。

### 合并小文件

　　我们知道文件数目小，容易在文件存储端造成瓶颈，给 HDFS 带来压力，影响处理效率。对此，可以通过合并 Map 和 Reduce 的结果文件来消除这样的影响。

　　用于设置合并属性的参数有：

*   是否合并 Map 输出文件：`hive.merge.mapfiles=true（默认值为真）`
*   是否合并 Reduce 端输出文件：`hive.merge.mapredfiles=false（默认值为假）`
*   合并文件的大小：`hive.merge.size.per.task=256*1000*1000（默认值为 256000000）`

补充：实际集群操作过程中，join 时候的**小表在前**的原则是比较先接触到的，这点在查阅一些资料和问过同事之后觉得是最快优化 join 操作的，而 mapjoin 则几乎没有用到，可能接触到的小表量级也是比较大的，而且公司的 hive 貌似是 0.12 的了，应该是自动优化的把

## 程序角度优化

### 熟练使用 SQL 提高查询

> 熟练地使用 SQL，能写出高效率的查询语句。

　　场景：有一张 user 表，为卖家每天收到表，user_id，ds（日期）为 key，属性有主营类目，指标有交易金额，交易笔数。每天要取前 10 天的总收入，总笔数，和最近一天的主营类目。 　　

**常用方法**

```sql
# 第一步：利用分析函数，取每个 user_id 最近一天的主营类目，存入临时表 t1。
CREATE TABLE t1 AS
SELECT 
    user_id,
    substr(MAX(CONCAT(ds,cat),9) AS main_cat) 
FROM users 
WHERE ds=20120329 // 20120329 为日期列的值，实际代码中可以用函数表示出当天日期 GROUP BY user_id; 

# 第二步：汇总 10 天的总交易金额，交易笔数，存入临时表 t2
CREATE TABLE t2 AS
SELECT 
    user_id,
    sum(qty) AS qty,SUM(amt) AS amt 
FROM users 
WHERE ds BETWEEN 20120301 AND 20120329 
GROUP BY user_id 

# 第三步：关联 t1，t2，得到最终的结果。
SELECT 
    t1.user_id,
    t1.main_cat,
    t2.qty,t2.amt 
FROM t1 
JOIN t2 ON t1.user_id=t2.user_id
```

**优化方法**　

```sql
SELECT 
    user_id,
    substr(MAX(CONCAT(ds,cat)),9) AS main_cat,
    SUM(qty),
    SUM(amt) 
FROM users 
WHERE ds BETWEEN 20120301 AND 20120329 
GROUP BY user_id
```

> 在工作中我们总结出：方案 2 的开销等于方案 1 的第二步的开销，性能提升，由原有的 25 分钟完成，缩短为 10 分钟以内完成。节省了两个临时表的读写是一个关键原因，这种方式也适用于 Oracle 中的数据查找工作。 SQL 具有普适性，很多 SQL 通用的优化方案在 Hadoop 分布式计算方式中也可以达到效果。

补充：实际集群操作过程中，第一种普通操作是要被同事嘲笑的，一般写 join 的复合类的操作，我们尽量将把它写在同一段代码中，所以可能会出现一段 hive 有七八个 join，只有当需要产出中间表或者业务逻辑有点混乱的时候，我们才存储中间表然后再重新写下，一般而言，我们抽取数据的时候使用核心表去 left join 其他表，这样就保证了核心表中的字段都会在，即使匹配不到也会存在 Null 而不是数据的丢失，这对于我们之后计算指标来说是比较重要的

### 无效 ID 在关联时的数据倾斜问题

> 问题：日志中常会出现信息丢失，比如每日约为 20 亿的全网日志，其中的 user_id 为主 键，在日志收集过程中会丢失，出现主键为 null 的情况，如果取其中的 user_id 和 bmw_users 关联，就会碰到数据倾斜的问题。原因是 Hive 中，主键为 null 值的项会被当做相同的 Key 而分配进同一个计算 Map。

*   解决方法 1：user_id 为空的不参与关联，子查询过滤 null

```sql
SELECT 
* 
FROM log a 
JOIN 
bmw_users b 
ON a.user_id IS NOT NULL AND a.user_id=b.user_id 
UNION All 
SELECT 
* 
FROM log a 
WHERE a.user_id IS NULL
```

*   解决方法 2: user_id 为空时添加随机数

```sql
SELECT 
* 
FROM log a 
LEFT OUTER JOIN 
bmw_users b 
ON 
CASE WHEN a.user_id IS NULL THEN CONCAT('dp_hive',RAND()) ELSE a.user_id END =b.user_id; 
```

> 调优结果：原先由于数据倾斜导致运行时长超过 1 小时，解决方法 1 运行每日平均时长 25 分钟，解决方法 2 运行的每日平均时长在 20 分钟左右。优化效果很明显。

　　我们在工作中总结出：解决方法 2 比解决方法 1 效果更好，不但 IO 少了，而且作业数也少了。解决方法 1 中 log 读取两次，job 数为 2。解决方法 2 中 job 数是 1。这个优化适合无效 id（比如 - 99、 ‘’，null 等）产生的倾斜问题。**把空值的 key 变成一个字符串加上随机数，就能把倾斜的 数据分到不同的 Reduce 上，从而解决数据倾斜问题**。因为空值不参与关联，即使分到不同 的 Reduce 上，也不会影响最终的结果。附上 Hadoop 通用关联的实现方法是：关联通过二次排序实现的，关联的列为 partion key，关联的列和表的 tag 组成排序的 group key，根据 pariton key 分配 Reduce。同一 Reduce 内根据 group key 排序。

### 不同数据类型关联产生的倾斜问题

> 问题：不同数据类型 id 的关联会产生数据倾斜问题。

 一张表 s8 的日志，每个商品一条记录，要和商品表关联。但关联却碰到倾斜的问题。 s8 的日志中有 32 为字符串商品 id，也有数值商品 id，日志中类型是 string 的，但商品中的 数值 id 是 bigint 的。猜想问题的原因是把 s8 的商品 id 转成数值 id 做 hash 来分配 Reduce， 所以字符串 id 的 s8 日志，都到一个 Reduce 上了，解决的方法验证了这个猜测。

*   解决方法：把数据类型转换成字符串类型

```sql
SELECT 
* 
FROM s8_log a 
LEFT OUTER JOIN 
r_auction_auctions b 
ON a.auction_id=CASE(b.auction_id AS STRING) 
```

> 调优结果显示：数据表处理由 1 小时 30 分钟经代码调整后可以在 20 分钟内完成。

补充：话说 hive 不是自带隐式转换么，需要将 bigint 类型强制转换成 string 类型？

### 利用 Hive 对 UNION ALL 优化的特性

> 问题：比如推广效果表要和商品表关联，效果表中的 auction_id 列既有 32 为字符串商 品 id，也有数字 id，和商品表关联得到商品的信息。

*   解决方法：union all

```sql
SELECT 
* 
FROM effect a 
JOIN 
(
  SELECT 
    auction_id AS auction_id 
  FROM auctions 
  UNION All 
  SELECT 
    auction_string_id AS auction_id 
  FROM auctions
)b 
ON a.auction_id=b.auction_id 
```

**多表 union all 会优化成一个 job。比分别过滤数字 id，字符串 id 然后分别和商品表关联性能要好。**

> 这样写的好处：1 个 MapReduce 作业，商品表只读一次，推广效果表只读取一次。把 这个 SQL 换成 Map/Reduce 代码的话，Map 的时候，把 a 表的记录打上标签 a，商品表记录 每读取一条，打上标签 b，变成两个

### 解决 Hive 对 UNION ALL 优化的短板

> Hive 对 union all 的优化的特性：**对 union all 优化只局限于非嵌套查询**。

#### 消灭子查询内的 group by

*   示例 1：子查询内有 group by

```sql
SELECT 
* 
FROM 
(
    SELECT 
      * 
    FROM t1 
    GROUP BY c1,c2,c3 
UNION ALL 
    SELECT
      * 
    FROM t2 
    GROUP BY c1,c2,c3
)t3 
GROUP BY c1,c2,c3 
```

> 从业务逻辑上说，子查询内的 GROUP BY 怎么都看显得多余（功能上的多余，除非有 COUNT(DISTINCT)），如果不是因为 Hive Bug 或者性能上的考量（曾经出现如果不执行子查询 GROUP BY，数据得不到正确的结果的 Hive Bug）。所以这个 Hive 按经验转换成如下所示：

```sql
SELECT 
* 
FROM 
(
    SELECT 
    * 
    FROM t1 
  UNION ALL 
    SELECT 
    * 
    FROM t2
)t3 
GROUP BY c1,c2,c3 
```

　　调优结果：经过测试，并未出现 union all 的 Hive Bug，数据是一致的。MapReduce 的 作业数由 3 减少到 1。

 t1 相当于一个目录，t2 相当于一个目录，对 Map/Reduce 程序来说，t1，t2 可以作为 Map/Reduce 作业的 mutli inputs。这可以通过一个 Map/Reduce 来解决这个问题。Hadoop 的 计算框架，不怕数据多，就怕作业数多。

　　但如果换成是其他计算平台如 Oracle，那就不一定了，因为把大的输入拆成两个输入， 分别排序汇总后 merge（假如两个子排序是并行的话），是有可能性能更优的（比如希尔排 序比冒泡排序的性能更优）。

#### 消灭子查询内的 COUNT(DISTINCT)，MAX，MIN。

```sql
SELECT 
* 
FROM 
(
    SELECT 
    * 
    FROM t1 
  UNION ALL 
    SELECT 
      c1,
      c2,
      c3,
      COUNT(DISTINCT c4) 
    FROM t2 
    GROUP BY c1,c2,c3
)t3 
GROUP BY c1,c2,c3; 
```

　　由于子查询里头有 COUNT(DISTINCT) 操作，直接去 GROUP BY 将达不到业务目标。**这时采用临时表消灭 COUNT(DISTINCT) 作业不但能解决倾斜问题，还能有效减少 jobs。**

```sql
INSERT t4 SELECT c1,c2,c3,c4 FROM t2 GROUP BY c1,c2,c3; 

SELECT 
c1,
c2,
c3,
SUM(income),
SUM(uv) 
FROM 
(
    SELECT 
      c1,
      c2,
      c3,
      income,
      0 AS uv 
    FROM t1 
  UNION ALL 
    SELECT 
      c1,
      c2,
      c3,
      0 AS income,
      1 AS uv 
    FROM t2
)t3 
GROUP BY c1,c2,c3;
```

　　job 数是 2，减少一半，而且两次 Map/Reduce 比 COUNT(DISTINCT) 效率更高。

 调优结果：千万级别的类目表，member 表，与 10 亿级得商品表关联。原先 1963s 的任务经过调整，1152s 即完成。

#### 消灭子查询内的 JOIN

```sql
SELECT 
* 
FROM 
(
      SELECT 
        * 
      FROM t1 
    UNION ALL 
      SELECT 
        * 
      FROM t4 
    UNION ALL 
      SELECT 
        * 
      FROM t2 
      JOIN t3 
      ON t2.id=t3.id
)x 
GROUP BY c1,c2; 
```

> 上面代码运行会有 5 个 jobs。加入先 JOIN 生存临时表的话 t5，然后 UNION ALL，会变成 2 个 jobs。

```sql
INSERT OVERWRITE TABLE t5 
SELECT * FROM t2 JOIN t3 ON t2.id=t3.id; 

SELECT * FROM (t1 UNION ALL t4 UNION ALL t5); 
```

　　调优结果显示：针对千万级别的广告位表，由原先 5 个 Job 共 15 分钟，分解为 2 个 job 一个 8-10 分钟，一个 3 分钟。

补充：第二个消灭子查询内的 JOIN，我觉得意义不是很大，第一需要建立临时表，这个内表还需要定时清理，第二没有从原理上进行优化，只是把步骤分开了而已，所以意义并非那么大，如果说从减少 job 的意义上来说，的确有提升，但是如果是业务代码块，感觉分开写意义不是很大，这是我个人的理解

### GROUP BY 替代 COUNT(DISTINCT) 达到优化效果

> 计算 uv 的时候，经常会用到 COUNT(DISTINCT)，但在数据比较倾斜的时候 COUNT(DISTINCT) 会比较慢。这时可以尝试用 GROUP BY 改写代码计算 uv。

*   原有代码

```sql
ALTER  TABLE s_dw_tanx_adzone_uv ADD PARTITION (ds=20120329) 
SELECT 
  20120329 AS thedate,
  adzoneid,
  COUNT(DISTINCT acookie) AS uv 
FROM s_ods_log_tanx_pv t 
WHERE t.ds=20120329 
GROUP BY adzoneid
```

关于 COUNT(DISTINCT) 的数据倾斜问题不能一概而论，要依情况而定，下面是我测试的一组数据：

测试数据：169857 条

```sql
#统计每日IP 
CREATE TABLE ip_2014_12_29 AS 
SELECT 
COUNT(DISTINCT ip) AS IP 
FROM logdfs 
WHERE logdate='2014_12_29'; 
耗时：24.805 seconds 

#统计每日IP（改造） 
CREATE TABLE ip_2014_12_29 AS 
SELECT
COUNT(1) AS IP 
FROM 
(
SELECT 
DISTINCT ip 
from logdfs 
WHERE logdate='2014_12_29'
)tmp; 
耗时：46.833 seconds
```

> 测试结果表名：明显改造后的语句比之前耗时，这是因为改造后的语句有 2 个 SELECT，多了一个 job，这样在数据量小的时候，数据不会存在倾斜问题。

补充：相当于多做子查询操作，那肯定是变慢的

* * *

# 优化总结

　　优化时，把 hive sql 当做 mapreduce 程序来读，会有意想不到的惊喜。理解 hadoop 的核心能力，是 hive 优化的根本。这是这一年来，项目组所有成员宝贵的经验总结。

> 长期观察 hadoop 处理数据的过程，有几个显著的特征:

1.  不怕数据多，就怕数据倾斜。
2.  对 jobs 数比较多的作业运行效率相对比较低，比如即使有几百行的表，如果多次关联多次汇总，产生十几个 jobs，没半小时是跑不完的。map reduce 作业初始化的时间是比较长的。
3.  对 sum，count 来说，不存在数据倾斜问题。
4.  对 count(distinct), 效率较低，数据量一多，准出问题，如果是多 count(distinct ) 效率更低。

> 优化可以从几个方面着手：

1.  好的模型设计事半功倍。
2.  解决数据倾斜问题。
3.  减少 job 数。
4.  设置合理的 map reduce 的 task 数，能有效提升性能。(比如，10w + 级别的计算，用 160 个 reduce，那是相当的浪费，1 个足够)。
5.  自己动手写 sql 解决数据倾斜问题是个不错的选择。set hive.groupby.skewindata=true; 这是通用的算法优化，但算法优化总是漠视业务，习惯性提供通用的解决方法。 Etl 开发人员更了解业务，更了解数据，所以通过业务逻辑解决倾斜的方法往往更精确，更有效。
6.  对 count(distinct) 采取漠视的方法，尤其数据大的时候很容易产生倾斜问题，不抱侥幸心理。自己动手，丰衣足食。
7.  对小文件进行合并，是行至有效的提高调度效率的方法，假如我们的作业设置合理的文件数，对云梯的整体调度效率也会产生积极的影响。

**优化时把握整体，单个作业最优不如整体最优。**

* * *

# 更新

> 这是一个积累的过程，所以以后必然会有更新

*   2017.07.29 第一次更新
*   2017.08.04 第二次更新


# 致谢

*   [Hive 性能优化](http://www.cnblogs.com/smartloli/p/4356660.html)

*   [Hive 之 insert 和 insert overwrite](http://blog.chinaunix.net/uid-30041424-id-5763141.html)

*   [split 和 block 的区别以及 maptask 和 reducetask 个数设定](http://blog.csdn.net/qq_20641565/article/details/53457622)

*   [hadoop 中，combine、partition、shuffle 作用分别是什么？](http://www.aboutyun.com/thread-7104-1-1.html)

*   [Hive 的 HQL 语句及数据倾斜解决方案](http://blog.csdn.net/sdksdk0/article/details/51675005)

*   [Hive - hive.groupby.skewindata 环境变量与负载均衡](http://blog.csdn.net/evo_steven/article/details/17526725)

*   [深入浅出数据仓库中 SQL 性能优化之 Hive 篇](http://www.aboutyun.com/forum.php?mod=viewthread&tid=11349)
