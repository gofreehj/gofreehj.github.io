---
title: Hive中集合数据类型Struct，Map和Array
date: 2019-07-07 12:33:48
tags: 
    - sql
categories: 
    - Hive
---

Hive中的列支持使用struct，map和array集合数据类型。下表中的数据类型实际上调用的是内置函数。

Hive集合数据类型:

| **数据类型** | **描述**                                                     | **字面语法示例**                 |
| ------------ | ------------------------------------------------------------ | -------------------------------- |
| STRUCT       | 数据类型描述字面语法示例和C语言中的struct或者“对象”类似，都可以通过“点”符号访问元素内容。例如，如果某个列的数据类型是 STRUCT { first STRING , last STRING} ，那么第 1 个元素可以通过字段名.first来引用 | struct('John','Doe')             |
| MAP          | MAP 是一组键一值对元组集合，使用数组表示法(例如['key']) 可以访问元素。例如，如果某个列的数据类型是 MAP ，其中键 值对是'first' -> 'John' 和'last' -> 'Doe'，那么可以通过字段名['last']获取最后 1 个元素 | map('first','JOIN','last','Doe') |
| ARRAY        | 数组是一组具有相同类型和名称的变量的集合。这些变量称为数组的元素，每个数组元素都有一个编号，编号从零开始。例如，数组值为［'John', 'Doe'] , 那么第 2 个元素可以通过数组名[1]进行引用 | Array('John','Doe')              |

和基本数据类型一样，这些类型的名称同样是保留字。

大多数的关系型数据库并不支持这些集合数据类型，因此使用它们会趋向于破坏标准格式。例如，在传统数据模型中，structs可能需要由多个不同的表拼装而成，表间需要适当地使用外键来进行连接。

破坏标准格式所带来的一个实际问题是会增大数据冗余的风险，进而导致消耗不必要的磁盘空间，还有可能造成数据不一致，因此当数据发生改变时冗余的拷贝数据可能无法进行相应的同步。

然而，在大数据系统中，不遵循标准格式的一个好处就是可以提高更高吞吐量的数据。当处理的数据的数量级是TB或者PB时，以最少的“头部寻址”来从磁盘上扫描数据是非常必要的。按数据进行封装的话可以通过减少寻址次数来提供查询的速度。而如果根据外键关系关联的话则需要进行磁盘间的寻址操作，这样会有非常高的性能消耗。

建表：

```sql
create
	table collect_test
	(
		id INT,
		name STRING,
		hobby ARRAY < STRING >,                      -- array中元素为String类型
		friend MAP < STRING,STRING >,                -- map中键和值均为String类型
		mark struct < math:int,english:int >         -- Struct中元素为Int类型
	)
	row format delimited fields terminated by ','  -- 字段之间用','分隔
	collection items terminated by '_'             -- 集合中的元素用'_'分隔
	map keys terminated by ':'                     -- map中键值对之间用':'分隔
	lines terminated by '\n                        -- 行之间用'\n'分隔 默认一般不指定
```

2、向表test_set中插入数据

1）对于数据量较大，常用的一种方法是通过文件批量导入的方法，比如我现在要将如下的文本中的数据插入到表中

```sql
1,xiaoming,basketball_game,xiaohong:yes_xiaohua:no,99_75

1,xiaohong,watch_study,xiaoming:no_xiaohua:not,95_95
```

可以采用如下语句来实现

```sql
hive -e "load data local inpath '/path/data.txt' overwrite into table collect_test"
```

2）对于想插入几条数据时，可以采取insert语句来插入数据，比如我们想插入数据

2,xiaohua,basketball_read,xiaoming:no_xiaohong:no,90_90
可以采用如下语句来实现，分别通过array,str_to_map,named_struct来包装插入的三种集合数据

```sql
INSERT INTO collect_test
SELECT
	2,
	'xiaohua',
	array('basketball', 'read'),
	str_to_map('xiaoming:no,xiaohong:no'),
	named_struct('math', 90, 'english', 90)
```

对于集合类型的查询，我们还有一种经常使用的方法，查询语句如下

```sql
select
	id,
	name,
	hobby[0], 					-- 查询第一个hobby
	friend['xiaohong'], -- 查询map键为xiaohong的value
	mark.math 					-- 查询struct中math的值
from
	test_set
where
	name = 'xiaoming'
```

----

参考：

1.https://blog.csdn.net/qq_41973536/article/details/81627918

2.https://blog.csdn.net/u014414323/article/details/83616361