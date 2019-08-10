[TOC]

# MySQL表字段字符集不同导致的索引失效问题



故障症状：一位同学的MySQL机器上面发现了这样一个问题，MySQL两张表做left join时，执行计划里面显示有一张表使用了全表扫描，扫描全表近100万行记录，大并发的这样的SQL过来数据库变得几乎不可用了。MySQL版本为官方5.7.12。

 故障原因：字符集不同引起的

##  问题重现

首先，表结构和表记录如下：

```
mysql> show create table t1\G 
*************************** 1. row *************************** 
Table: t1 Create Table: CREATE TABLE `t1` ( 
`id` int(11) NOT NULL AUTO_INCREMENT, `name` varchar(20) DEFAULT NULL, 
`code` varchar(50) DEFAULT NULL, 
PRIMARY KEY (`id`), KEY `idx_code` (`code`), 
KEY `idx_name` (`name`) ) ENGINE=InnoDB AUTO_INCREMENT=6 
DEFAULT CHARSET=utf8 1 row in set (0.00 sec)`
 
mysql> show create table t2\G
 *************************** 1. row ***************************
 Table: t2
 Create Table: CREATE TABLE `t2` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `name` varchar(20) DEFAULT NULL,
 `code` varchar(50) DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `idx_code` (`code`),
 KEY `idx_name` (`name`)
 ) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8mb4
 1 row in set (0.00 sec)
 
mysql> select * from t1;
 +----+------+----------------------------------+
 | id | name | code |
 +----+------+----------------------------------+
 | 1 | aaaa | 0752b0e3c72d4f5c701728db8ea8a3f9 |
 | 2 | bbbb | 36d8147db18d55e64c8b5ea8679328b7 |
 | 3 | cccc | dc3bab5197eeb6b315204f0af563c961 |
 | 4 | dddd | 1bb4dc313a54e4c0ee04644d2a1fe900 |
 | 5 | eeee | f33180d7745079d2dfaaace2fdd74b2a |
 +----+------+----------------------------------+
 5 rows in set (0.00 sec)
 
mysql> select * from t2;
 +----+------+----------------------------------+
 | id | name | code |
 +----+------+----------------------------------+
 | 1 | aaaa | bca3bc1eb999136d6e6f877d9accc918 |
 | 2 | bbbb | 77dd5d07ea1c458afd76c8a6d953cf0a |
 | 3 | cccc | 3ac617d1857444e5383f074c60af7efd |
 | 4 | dddd | 8a77a32a7e0825f7c8634226105c42e5 |
 | 5 | eeee | 0c7fc18b8995e9e31ca774b1312be035 |
 +----+------+----------------------------------+
 5 rows in set (0.00 sec)
```

2张表left join的执行计划如下：

```SQL
mysql> desc select * from t2 left join t1 on t1.code = t2.code where t2.name = 'dddd'\G 
*************************** 1. row *************************** 
id: 1 select_type: SIMPLE 
table: t2 
partitions: NULL 
type: ref 
possible_keys: idx_name 
key: idx_name 
key_len: 83 
ref: const 
rows: 1 
filtered: 100.00 
Extra: NULL 
*************************** 2. row *************************** 
id: 1 
select_type: SIMPLE 
table: t1 
partitions: NULL 
type: ALL 
possible_keys: NULL 
key: NULL 
key_len: NULL 
ref: NULL 
rows: 5 
filtered: 100.00 
Extra: Using where; Using join buffer (Block Nested Loop) 
2 rows in set, 1 warning (0.01 sec)
```



可以明显地看到，[t2.name](http://t2.name/) = 'dddd'使用了索引，而t1.code = t2.code这个关联条件没有使用到t1.code上面的索引，一开始Scott也百思不得其解，但是机器不会骗人。Scott用show warnings查看改写后的执行计划如下：

```
mysql> show warnings; 
 /* select#1 */ select `testdb`.`t2`.`id` AS `id`,`testdb`.`t2`.`name` AS `name`,
 `testdb`.`t2`.`code` AS `code`,
 `testdb`.`t1`.`id` AS `id`,
 `testdb`.`t1`.`name` AS `name`,
 `testdb`.`t1`.`code` AS `code` 
 from `testdb`.`t2` left join `testdb`.`t1` 
 on((convert(`testdb`.`t1`.`code` using utf8mb4) = `testdb`.`t2`.`code`))
 where (`testdb`.`t2`.`name` = 'dddd') 
```

在发现了convert(testdb.t1.code using utf8mb4)之后，Scott发现2个表的字符集不一样。t1为utf8，t2为utf8mb4。但是为什么表字符集不一样（实际是字段字符集不一样）就会导致t1全表扫描呢？下面来做分析。

（1）首先t2 left join t1决定了t2是驱动表，这一步相当于执行了select * from t2 where [t2.name](http://t2.name/) = 'dddd'，取出code字段的值，这里为'8a77a32a7e0825f7c8634226105c42e5';

（2）然后拿t2查到的code的值根据join条件去t1里面查找，这一步就相当于执行了select * from t1 where t1.code = '8a77a32a7e0825f7c8634226105c42e5';

（3）但是由于第（1）步里面t2表取出的code字段是utf8mb4字符集，而t1表里面的code是utf8字符集，这里需要做字符集转换，字符集转换遵循由小到大的原则，因为utf8mb4是utf8的超集，所以这里把utf8转换成utf8mb4，即把t1.code转换成utf8mb4字符集，转换了之后，由于t1.code上面的索引仍然是utf8字符集，所以这个索引就被执行计划忽略了，然后t1表只能选择全表扫描。更糟糕的是，如果t2筛选出来的记录不止1条，那么t1就会被全表扫描多次，性能之差可想而知。

## 故障分类：查询语句

 故障解决方案：

既然原因已经清楚了，如何解决呢？当然是改字符集了，把t1改成和t2一样或者把t2改成t1都可以，这里选择把t1转成utf8mb4。那怎么转字符集呢？

有的同学会说用alter table t1 charset utf8mb4;但这是错的，这只是改了表的默认字符集，即新的字段才会使用utf8mb4，已经存在的字段仍然是utf8。

```
mysql> alter table t1 charset utf8mb4; Query OK, 0 rows affected (0.01 sec) Records: 0 Duplicates: 0 Warnings: 0
 
mysql> show create table t1\G
 *************************** 1. row ***************************
 Table: t1
 Create Table: CREATE TABLE `t1` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `name` varchar(20) CHARACTER SET utf8 DEFAULT NULL,
 `code` varchar(50) CHARACTER SET utf8 DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `idx_code` (`code`),
 KEY `idx_name` (`name`)
 ) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8mb4
 1 row in set (0.00 sec)
```

只有用alter table t1 convert to charset utf8mb4;才是正确的。

但是还要注意一点，alter table 改字符集的操作是阻塞写的（用lock = node会报错）所以业务高峰时请不要操作，即使在业务低峰时期，大表的操作仍然建议使用pt-online-schema-change在线修改字符集。

```SQL
mysql> alter table t1 convert to charset utf8mb4, lock=none; 
ERROR 1846 (0A000): LOCK=NONE is not supported. Reason: Cannot change column type INPLACE. Try LOCK=SHARED. 
mysql> alter table t1 convert to charset utf8mb4, lock=shared; Query OK, 5 rows affected (0.04 sec) Records: 5 Duplicates: 0 Warnings: 0
 
mysql> show create table t1\G
 *************************** 1. row ***************************
 Table: t1
 Create Table: CREATE TABLE `t1` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `name` varchar(20) DEFAULT NULL,
 `code` varchar(50) DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `idx_code` (`code`),
 KEY `idx_name` (`name`)
 ) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8mb4
 1 row in set (0.00 sec)
```

现在再来查看执行计划，可以看到已经没问题了。

 



```SQL
mysql> desc select * from t2 join t1 on t1.code = t2.code where t2.name = 'dddd'\G；
*************************** 1. row *************************** 
id: 1 select_type: SIMPLE 
table: t2 
partitions: NULL type: ref 
possible_keys: idx_code,idx_name 
key: idx_name key_len: 83 
ref: const 
rows: 1 
filtered: 100.00 
Extra: Using where 
*************************** 2. row *************************** 
id: 1 select_type: SIMPLE 
table: t1 partitions: NULL 
type: ref 
possible_keys: idx_code 
key: idx_code 
key_len: 203 
ref: testdb.t2.code
rows: 1 
filtered: 100.00 
Extra: NULL 
2 rows in set, 1 warning (0.00 sec)
```

## 注意点

 

（1）表字符集不同时，可能导致join的SQL使用不到索引，引起严重的性能问题；

（2）SQL上线前要做好SQL Review工作，尽量在和生产环境一样的环境下Review；

（3）改字符集的alter table操作会阻塞写，尽量在业务低峰操作，建议用pt-online-schema-change;

（4）表结构字符集要保持一致，发布时要做好审核工作；

（5）如果要大批量修改表的字符集，同样做好SQL的Review工作，关联的表的字符集一起做修改。