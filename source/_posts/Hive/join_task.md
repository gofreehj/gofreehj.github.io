---
title: Hive的三种Join方式
date: 2019-07-09 12:33:48
tags: 
    - sql
categories: 
    - Hive
---

Hive中就是把Map，Reduce的Join拿过来，通过SQL来表示。
参考链接：https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Joins

### Common/Shuffle/Reduce Join

假设要进行join的数据分别来自File1和File2

common join是一种最简单的join方式，其主要思想如下：

在map阶段，map函数同时读取两个文件File1和File2，为了区分两种来源的key/value数据对，对每条数据打一个标签 （tag）,比如：tag=0表示来自文件File1，tag=2表示来自文件File2。即：map阶段的主要任务是对不同文件中的数据打标签。

在reduce阶段，reduce函数获取key相同的来自File1和File2文件的value list， 然后对于同一个key，对File1和File2中的数据进行join（笛卡尔乘积）。即：reduce阶段进行实际的连接操作。

![](https://raw.githubusercontent.com/gofreehj/BigData/master/Hive/images/reduce_join.png)



![shuffle](https://raw.githubusercontent.com/gofreehj/BigData/master/Hive/images/shuffle.jpg)

### Map Join

之所以存在reduce side join，是因为在map阶段不能获取所有需要的join字段，即：同一个key对应的字段可能位于不同map中。Reduce side join是非常低效的，因为shuffle阶段要进行大量的数据传输。

Map side join是针对以下场景进行的优化：两个待连接表中，有一个表非常大，而另一个表非常小，以至于小表可以直接存放到内存中。这样，我们可以将小表复制多 份，让每个map task内存中存在一份（比如存放到hash table中），然后只扫描大表：对于大表中的每一条记录key/value，在hash table中查找是否有相同的key的记录，如果有，则连接后输出即可。

为了支持文件的复制，Hadoop提供了一个类DistributedCache，使用该类的方法如下：

（1）用户使用静态方法DistributedCache.addCacheFile()指定要复制的文件，它的参数是文件的URI（如果是 HDFS上的文件，可以这样：hdfs://namenode:9000/home/XXX/file，其中9000是自己配置的NameNode端口 号）。JobTracker在作业启动之前会获取这个URI列表，并将相应的文件拷贝到各个TaskTracker的本地磁盘上。（2）用户使用 DistributedCache.getLocalCacheFiles()方法获取文件目录，并使用标准的文件读写API读取相应的文件。

![map_join](https://raw.githubusercontent.com/gofreehj/BigData/master/Hive/images/map_join.jpg)

1） 大小表连接：

如果一张表的数据很大，另外一张表很少(<1000行)，那么我们可以将数据量少的那张表放到内存里面，在map端做join。 Hive支持Map Join，用法如下

```sql
select /*+ MAPJOIN(time_dim) */ count(1) from
store_sales join time_dim on (ss_sold_time_sk = t_time_sk)
```



2） 需要做不等值join操作（a.x < b.y 或者 a.x like b.y等）

这种操作如果直接使用join的话语法不支持不等于操作，hive语法解析会直接抛出错误 如果把不等于写到where里会造成笛卡尔积，数据异常增大，速度会很慢。甚至会任务无法跑成功~ 根据mapjoin的计算原理，MapJoin会把小表全部读入内存中，在map阶段直接拿另外一个表的数据和内存中表数据做匹配。这种情况下即使笛卡尔积也不会对任务运行速度造成太大的效率影响。 而且hive的where条件本身就是在map阶段进行的操作，所以在where里写入不等值比对的话，也不会造成额外负担。

```sql
select /*+ MAPJOIN(a) */
a.start_level, b.*
from dim_level a
join (select * from test) b
where b.xx>=a.start_level and b.xx<end_level;
```

 3） MAPJOIN 结合 UNIONALL
原始sql：

```sql
select a.*,coalesce(c.categoryid,’NA’) as app_category
from (select * from t_aa_pvid_ctr_hour_js_mes1
) a
left outer join
(select * fromt_qd_cmfu_book_info_mes
) c
on a.app_id=c.book_id;
```

速度很慢，老办法，先查下数据分布:

```sql
select *
from
(selectapp_id,count(1) cnt
fromt_aa_pvid_ctr_hour_js_mes1
group by app_id) t
order by cnt DESC
limit 50;
```

数据分布如下：

```
NA      617370129
2       118293314
1       40673814
d       20151236
b       1846306
s       1124246
5       675240
8       642231
6       611104
t       596973
4       579473
3       489516
7       475999
9       373395
107580  10508
```



我们可以看到除了NA是有问题的异常值，还有appid=1~9的数据也很多，而这些数据是可以关联到的，所以这里不能简单的随机函数了。而fromt_qd_cmfu_book_info_mes这张app库表，又有几百万数据，太大以致不能放入内存使用mapjoin。

解决方：首先将appid=NA和1到9的数据存入一组，并使用mapjoin与维表（维表也限定appid=1~9，这样内存就放得下了）关联，而除此之外的数据存入另一组，使用普通的join，最后使用union all 放到一起。

```sql
select a.*,coalesce(c.categoryid,’NA’) as app_category
from --if app_id isnot number value or <=9,then not join
(select * fromt_aa_pvid_ctr_hour_js_mes1
where cast(app_id asint)>9
) a
left outer join
(select * fromt_qd_cmfu_book_info_mes
where cast(book_id asint)>9) c
on a.app_id=c.book_id
union all
select /*+ MAPJOIN(c)*/
a.*,coalesce(c.categoryid,’NA’) as app_category
from –if app_id<=9,use map join
(select * fromt_aa_pvid_ctr_hour_js_mes1
where coalesce(cast(app_id as int),-999)<=9) a
left outer join
(select * fromt_qd_cmfu_book_info_mes
where cast(book_id asint)<=9) c
--if app_id is notnumber value,then not join
on a.app_id=c.book_id
```

- 设置：

  当然也可以让hive自动识别，把join变成合适的Map Join如下所示 注：当设置为true的时候，hive会自动获取两张表的数据，判定哪个是小表，然后放在内存中

```sql
set hive.auto.convert.join=true;
select count(*) from store_sales join time_dim on (ss_sold_time_sk = t_time_sk)
```

### SMB(Sort-Merge-Buket) Join

- 场景：

  大表对小表应该使用MapJoin，但是如果是大表对大表，如果进行shuffle，那就要人命了啊，第一个慢不用说，第二个容易出异常，既然是两个表进行join，肯定有相同的字段吧。

  tb_a - 5亿（按排序分成五份，每份1亿放在指定的数值范围内,类似于分区表） a_id 100001 ~ 110000 - bucket-01-a -1亿 110001 ~ 120000 120001 ~ 130000 130001 ~ 140000 140001 ~ 150000

  tb_b - 5亿（同上，同一个桶只能和对应的桶内数据做join） b_id 100001 ~ 110000 - bucket-01-b -1亿 110001 ~ 120000 120001 ~ 130000 130001 ~ 140000 140001 ~ 150000

  *注：实际生产环境中，一天的数据可能有50G（举例子可以把数据弄大点，比如说10亿分成1000个bucket）。*

- 原理：

  在运行SMB Join的时候会重新创建两张表，当然这是在后台默认做的，不需要用户主动去创建，如下所示：

  ![SMB Join](https://raw.githubusercontent.com/gofreehj/BigData/master/Hive/images/smb_join.png)

**设置（默认是false）：**

```sql
set hive.auto.convert.sortmerge.join=true
set hive.optimize.bucketmapjoin=true;
set hive.optimize.bucketmapjoin.sortedmerge=true;
```

- 总结：

  其实在写程序的时候，我们就可以知道哪些是大表哪些是小表，注意调优。

  

  任务执行计划参见 ： Map join和Common join详解



----

参考：

[Hive Join的实现原理](https://blog.csdn.net/u013668852/article/details/79768266)

[Hive的三种Join方式](https://www.cnblogs.com/raymoc/p/5323824.html)

[Map join和Common join详解](https://blog.csdn.net/weixin_39216383/article/details/79043299)

[SQL join中级篇--hive中 mapreduce join方法分析](https://www.cnblogs.com/chengyeliang/p/4512207.html)