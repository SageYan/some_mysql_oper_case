[TOC]

# MySQL FlashBack 操作规范

![](C:\Users\yancong\Desktop\image2018-10-10_17-11-23.png)

## 1.1 与客户沟通故障时间

- 误操作人只能知道大致的误操作时间
- 根据大致时间过滤数据
- 根据数据量取误操作前后多长时间（默认10分钟）

## 1.2 与客户沟通误操作的内容

- 误操作表
- 误操作执行语句

## 1.3 将执行服务器ECS允许连接MySQL

- 自建数据库授权
- RDS添加白名单

## 1.4 利用binlog2sql工具解析出标准SQL

- 获取误操作SQL执行具体时间
- 获取误操作SQL执行具体位置

## 1.5 利用binlog2sql工具解析出回滚SQL

- 利用回滚SQL恢复数据

# 二、binlog2sql工具

从MySQL binlog解析出你要的SQL。根据不同选项，你可以得到原始SQL、回滚SQL、去除主键的INSERT SQL等。

## 2.1 用途

- 数据快速回滚（闪回）
- 主从切换后master丢数据的修复
- 从binlog生成标准SQL，带来的衍生功能

## 2.2 限制

- mysql server必须开启，离线模式下不能解析
- 参数 binlog_row_image 必须为FULL，暂不支持MINIMAL
- 解析速度不如mysqlbinlog

## 2.3 安装

```SQL
--安装
shell> git clone https://github.com/danfengcao/binlog2sql.git && cd binlog2sql
[root@mysql1 binlog2sql]# ll
总用量 52
drwxr-xr-x. 2 root root   95 7月 28 13:59 binlog2sql
drwxr-xr-x. 2 root root   53 7月 28 13:39 example
-rw-r--r--. 1 root root 35141 7月 28 13:39 LICENSE
-rw-r--r--. 1 root root 9110 7月 28 13:39 README.md
-rw-r--r--. 1 root root   54 7月 28 13:39 requirements.txt
drwxr-xr-x. 2 root root   36 7月 28 13:39 tests
 
[root@mysql1 binlog2sql]# yum -y install epel-release
[root@mysql1 binlog2sql]# yum -y install python-pip
[root@mysql1 binlog2sql]# yum clean all
[root@mysql1 binlog2sql]# pip install --upgrade pip
 
shell> pip install -r requirements.txt
 
[root@mysql1 binlog2sql]# cat requirements.txt 
PyMySQL==0.7.11
wheel==0.29.0
mysql-replication==0.13
[root@mysql1 binlog2sql]# pip install -r requirements.txt 
Requirement already satisfied: PyMySQL==0.7.11 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 1))
Requirement already satisfied: wheel==0.29.0 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 2))
Requirement already satisfied: mysql-replication==0.13 in /usr/lib/python2.7/site-packages (from -r requirements.txt (line 3))
```

## 2.4使用

MySQL server必须设置以下参数:

```
[mysqld]
server_id = 1
log_bin = /var/log/mysql/mysql-bin.log
max_binlog_size = 1G
binlog_format = row
binlog_row_image = full
```

user需要的最小权限集合：

```
 
select, super/replication client, replication slave
 
建议授权
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 
```

## 2.5 binlog2sql帮助

```SQL
mysql连接配置
 
-h host; -P port; -u user; -p password
 
解析模式
 
--stop-never 持续同步binlog。可选。不加则同步至执行命令时最新的binlog位置。
 
-K, --no-primary-key 对INSERT语句去除主键。可选。
 
-B, --flashback 生成回滚语句，可解析大文件，不受内存限制，每打印一千行加一句SLEEP SELECT(1)。可选。与stop-never或no-primary-key不能同时添加。
 
解析范围控制
 
--start-file 起始解析文件。必须。
 
--start-position/--start-pos start-file的起始解析位置。可选。默认为start-file的起始位置。
 
--stop-file/--end-file 末尾解析文件。可选。默认为start-file同一个文件。若解析模式为stop-never，此选项失效。
 
--stop-position/--end-pos stop-file的末尾解析位置。可选。默认为stop-file的最末位置；若解析模式为stop-never，此选项失效。
 
--start-datetime 从哪个时间点的binlog开始解析，格式必须为datetime，如'2016-11-11 11:11:11'。可选。默认不过滤。
 
--stop-datetime 到哪个时间点的binlog停止解析，格式必须为datetime，如'2016-11-11 11:11:11'。可选。默认不过滤。
 
对象过滤
 
-d, --databases 只输出目标db的sql。可选。默认为空。
 
-t, --tables 只输出目标tables的sql。可选。默认为空。
```

# 三、案例

## 3.1 模拟业务场景

```SQL
 
--# 创建数据库
root@mysqldb 13:56: [(none)]> create database hjxdb;
Query OK, 1 row affected (0.00 sec)
 
--# 创建数据表
root@mysqldb 13:56: [hjxdb]> CREATE TABLE `user` (
   -> `id` bigint(20) unsigned NOT NULL DEFAULT '0' COMMENT '用户唯一标志符 UID',
   -> `username` varchar(64) DEFAULT NULL COMMENT '用户名，不区分大小写',
   -> `email` varchar(128) DEFAULT NULL COMMENT '注册邮箱，不区分大小写',
   -> `cell_phone` bigint(11) DEFAULT NULL COMMENT '手机号码',
   -> `password` char(32) NOT NULL COMMENT '密码hash值',
   -> `school_code` bigint(20) unsigned DEFAULT NULL COMMENT '学校代码',
   -> `register_time` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '注册时间',
   -> `usertype` int(5) NOT NULL DEFAULT '1' COMMENT '1为微信关注用户，2为微信登录app的用户，3为APP端微信登录的微信用户',
   -> `state` tinyint(4) NOT NULL DEFAULT '1' COMMENT '1<0: UID是否有效 1<1: 是否设置用户名密码 1<2: 是否认证邮箱 1<3: 是否认证手机号码',
   -> PRIMARY KEY (`id`)
   -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.12 sec)
 
--# 插入数据
root@mysqldb 13:57: [hjxdb]> insert into user values (1,'apple','apple@123.com','12233334444','hahaha',123,current_time(),1,2);
Query OK, 1 row affected (0.11 sec)
 
root@mysqldb 13:58: [hjxdb]> insert into user values (2,'pple','pple@123.com','12233334445','hahaha',124,current_time(),1,2);
Query OK, 1 row affected (0.00 sec)
 
root@mysqldb 13:58: [hjxdb]> insert into user values (3,'ple','ple@123.com','12233334446','hahaha',125,current_time(),1,2);
Query OK, 1 row affected (0.00 sec)
 
root@mysqldb 13:58: [hjxdb]> insert into user values (4,'le','le@123.com','12233334447','hahaha',126,current_time(),1,2);
Query OK, 1 row affected (0.11 sec)
 
--查询数据
root@mysqldb 14:00: [hjxdb]> select * from user;
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
| id | username | email         | cell_phone | password | school_code | register_time       | usertype | state |
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
| 1 | apple   | apple@123.com | 12233334444 | hahaha   |         123 | 2017-09-14 13:58:03 |       1 |     2 |
| 2 | pple     | pple@123.com | 12233334445 | hahaha   |         124 | 2017-09-14 13:58:10 |       1 |     2 |
| 3 | ple     | ple@123.com   | 12233334446 | hahaha   |         125 | 2017-09-14 13:58:16 |       1 |     2 |
| 4 | le       | le@123.com   | 12233334447 | hahaha   |         126 | 2017-09-14 13:58:34 |       1 |     2 |
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
4 rows in set (0.11 sec)
```

## 3.2 误修改数据

```SQL
 
--# 误将所有记录cell_phone 值更改为15811111111
root@mysqldb 14:00: [hjxdb]> update user set cell_phone=15811111111;
Query OK, 4 rows affected (0.01 sec)
Rows matched: 4 Changed: 4 Warnings: 0
 
--# 查询数据
root@mysqldb 14:07: [hjxdb]> select * from user;
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
| id | username | email         | cell_phone | password | school_code | register_time       | usertype | state |
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
| 1 | apple   | apple@123.com | 15811111111 | hahaha   |         123 | 2017-09-14 13:58:03 |       1 |     2 |
| 2 | pple     | pple@123.com | 15811111111 | hahaha   |         124 | 2017-09-14 13:58:10 |       1 |     2 |
| 3 | ple     | ple@123.com   | 15811111111 | hahaha   |         125 | 2017-09-14 13:58:16 |       1 |     2 |
| 4 | le       | le@123.com   | 15811111111 | hahaha   |         126 | 2017-09-14 13:58:34 |       1 |     2 |
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
4 rows in set (0.00 sec)
 
--# 误操作记录其他字段被修改
root@mysqldb 14:09: [hjxdb]> update user set school_code=127 where id=4;
Query OK, 1 row affected (0.11 sec)
Rows matched: 1 Changed: 1 Warnings: 0
 
root@mysqldb 14:09: [hjxdb]> select * from user;
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
| id | username | email         | cell_phone | password | school_code | register_time       | usertype | state |
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
| 1 | apple   | apple@123.com | 15811111111 | hahaha   |         123 | 2017-09-14 13:58:03 |       1 |     2 |
| 2 | pple     | pple@123.com | 15811111111 | hahaha   |         124 | 2017-09-14 13:58:10 |       1 |     2 |
| 3 | ple     | ple@123.com   | 15811111111 | hahaha   |         125 | 2017-09-14 13:58:16 |       1 |     2 |
| 4 | le       | le@123.com   | 15811111111 | hahaha   |         127 | 2017-09-14 13:58:34 |       1 |     2 |
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
4 rows in set (0.00 sec)
```

## 3.3 binlog2sql 解析标准SQL

```SQL
--# 查看当前binlog
root@mysqldb 14:09: [hjxdb]> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |     3282 |
+------------------+-----------+
1 row in set (0.00 sec)
 
--# 解析标准SQL，取误操作前后10分钟
[root@appserver binlog2sql]# python binlog2sql.py -h127.0.0.1 -P3306 -uroot -dhjxdb -t user --start-file='mysql-bin.000001' --start-datetime='2017-09-14 13:50:00' --stop-datetime='2017-09-14 14:10:00'
USE hjxdb;
create database hjxdb;
USE hjxdb;
CREATE TABLE `user` (
`id` bigint(20) unsigned NOT NULL DEFAULT '0' COMMENT '用户唯一标志符 UID',
`username` varchar(64) DEFAULT NULL COMMENT '用户名，不区分大小写',
`email` varchar(128) DEFAULT NULL COMMENT '注册邮箱，不区分大小写',
`cell_phone` bigint(11) DEFAULT NULL COMMENT '手机号码',
`password` char(32) NOT NULL COMMENT '密码hash值',
`school_code` bigint(20) unsigned DEFAULT NULL COMMENT '学校代码',
`register_time` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '注册时间',
`usertype` int(5) NOT NULL DEFAULT '1' COMMENT '1为微信关注用户，2为微信登录app的用户，3为APP端微信登录的微信用户',
`state` tinyint(4) NOT NULL DEFAULT '1' COMMENT '1<0: UID是否有效 1<1: 是否设置用户名密码 1<2: 是否认证邮箱 1<3: 是否认证手机号码',
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
[root@appserver binlog2sql]# python binlog2sql.py -h127.0.0.1 -P3306 -uroot -dhjxdb -t user --start-file='mysql-bin.000001' --start-datetime='2017-09-14 14:16:00' --stop-datetime='2017-09-14 14:27:00'
 
UPDATE `hjxdb`.`user` SET `username`='apple', `cell_phone`=15811111111, `register_time`='2017-09-14 13:58:03', `school_code`=123, `email`='apple@123.com', `usertype`=1, `state`=2, `password`='hahaha', `id`=1 WHERE `username`='apple' AND `cell_phone`=12233334444 AND `register_time`='2017-09-14 13:58:03' AND `school_code`=123 AND `email`='apple@123.com' AND `usertype`=1 AND `state`=2 AND `password`='hahaha' AND `id`=1 LIMIT 1; #start 4946 end 5601 time 2017-09-14 14:25:54
UPDATE `hjxdb`.`user` SET `username`='pple', `cell_phone`=15811111111, `register_time`='2017-09-14 13:58:10', `school_code`=124, `email`='pple@123.com', `usertype`=1, `state`=2, `password`='hahaha', `id`=2 WHERE `username`='pple' AND `cell_phone`=12233334445 AND `register_time`='2017-09-14 13:58:10' AND `school_code`=124 AND `email`='pple@123.com' AND `usertype`=1 AND `state`=2 AND `password`='hahaha' AND `id`=2 LIMIT 1; #start 4946 end 5601 time 2017-09-14 14:25:54
UPDATE `hjxdb`.`user` SET `username`='ple', `cell_phone`=15811111111, `register_time`='2017-09-14 13:58:16', `school_code`=125, `email`='ple@123.com', `usertype`=1, `state`=2, `password`='hahaha', `id`=3 WHERE `username`='ple' AND `cell_phone`=12233334446 AND `register_time`='2017-09-14 13:58:16' AND `school_code`=125 AND `email`='ple@123.com' AND `usertype`=1 AND `state`=2 AND `password`='hahaha' AND `id`=3 LIMIT 1; #start 4946 end 5601 time 2017-09-14 14:25:54
UPDATE `hjxdb`.`user` SET `username`='le', `cell_phone`=15811111111, `register_time`='2017-09-14 13:58:34', `school_code`=126, `email`='le@123.com', `usertype`=1, `state`=2, `password`='hahaha', `id`=4 WHERE `username`='le' AND `cell_phone`=12233334447 AND `register_time`='2017-09-14 13:58:34' AND `school_code`=126 AND `email`='le@123.com' AND `usertype`=1 AND `state`=2 AND `password`='hahaha' AND `id`=4 LIMIT 1; #start 4946 end 5601 time 2017-09-14 14:25:54
UPDATE `hjxdb`.`user` SET `username`='le', `cell_phone`=15811111111, `register_time`='2017-09-14 13:58:34', `school_code`=127, `email`='le@123.com', `usertype`=1, `state`=2, `password`='hahaha', `id`=4 WHERE `username`='le' AND `cell_phone`=15811111111 AND `register_time`='2017-09-14 13:58:34' AND `school_code`=126 AND `email`='le@123.com' AND `usertype`=1 AND `state`=2 AND `password`='hahaha' AND `id`=4 LIMIT 1; #start 5632 end 5921 time 2017-09-14 14:26:17
 
具体时间和具体位置
#start 4946 end 5601 time 2017-09-14 14:25:54
#start 5632 end 5921 time 2017-09-14 14:26:17
```

> 以上信息可以获取到误操作执行具体时间和具体位置

## 3.4 binlog2sql解析出回滚SQL

```SQL
--# 解析回滚SQL
[root@appserver binlog2sql]# python binlog2sql.py -h127.0.0.1 -P3306 -uroot -dhjxdb -t user --start-file='mysql-bin.000001' --start-position=4946 --stop-position=5921 -B > /tmp/rollback1.sql
 
--# 查看回滚SQL
[root@appserver binlog2sql]# cat /tmp/rollback1.sql 
UPDATE `hjxdb`.`user` SET `username`='le', `cell_phone`=15811111111, `register_time`='2017-09-14 13:58:34', `school_code`=126, `email`='le@123.com', `usertype`=1, `state`=2, `password`='hahaha', `id`=4 WHERE `username`='le' AND `cell_phone`=15811111111 AND `register_time`='2017-09-14 13:58:34' AND `school_code`=127 AND `email`='le@123.com' AND `usertype`=1 AND `state`=2 AND `password`='hahaha' AND `id`=4 LIMIT 1; #start 5632 end 5921 time 2017-09-14 14:26:17
UPDATE `hjxdb`.`user` SET `username`='le', `cell_phone`=12233334447, `register_time`='2017-09-14 13:58:34', `school_code`=126, `email`='le@123.com', `usertype`=1, `state`=2, `password`='hahaha', `id`=4 WHERE `username`='le' AND `cell_phone`=15811111111 AND `register_time`='2017-09-14 13:58:34' AND `school_code`=126 AND `email`='le@123.com' AND `usertype`=1 AND `state`=2 AND `password`='hahaha' AND `id`=4 LIMIT 1; #start 4946 end 5601 time 2017-09-14 14:25:54
UPDATE `hjxdb`.`user` SET `username`='ple', `cell_phone`=12233334446, `register_time`='2017-09-14 13:58:16', `school_code`=125, `email`='ple@123.com', `usertype`=1, `state`=2, `password`='hahaha', `id`=3 WHERE `username`='ple' AND `cell_phone`=15811111111 AND `register_time`='2017-09-14 13:58:16' AND `school_code`=125 AND `email`='ple@123.com' AND `usertype`=1 AND `state`=2 AND `password`='hahaha' AND `id`=3 LIMIT 1; #start 4946 end 5601 time 2017-09-14 14:25:54
UPDATE `hjxdb`.`user` SET `username`='pple', `cell_phone`=12233334445, `register_time`='2017-09-14 13:58:10', `school_code`=124, `email`='pple@123.com', `usertype`=1, `state`=2, `password`='hahaha', `id`=2 WHERE `username`='pple' AND `cell_phone`=15811111111 AND `register_time`='2017-09-14 13:58:10' AND `school_code`=124 AND `email`='pple@123.com' AND `usertype`=1 AND `state`=2 AND `password`='hahaha' AND `id`=2 LIMIT 1; #start 4946 end 5601 time 2017-09-14 14:25:54
UPDATE `hjxdb`.`user` SET `username`='apple', `cell_phone`=12233334444, `register_time`='2017-09-14 13:58:03', `school_code`=123, `email`='apple@123.com', `usertype`=1, `state`=2, `password`='hahaha', `id`=1 WHERE `username`='apple' AND `cell_phone`=15811111111 AND `register_time`='2017-09-14 13:58:03' AND `school_code`=123 AND `email`='apple@123.com' AND `usertype`=1 AND `state`=2 AND `password`='hahaha' AND `id`=1 LIMIT 1; #start 4946 end 5601 time 2017-09-14 14:25:54
 
--# 数据恢复
[root@appserver log]# mysql hjxdb < /tmp/rollback1.sql
[root@appserver log]# mysql
Welcome to the MariaDB monitor. Commands end with ; or \g.
Your MySQL connection id is 75
Server version: 5.6.35-log MySQL Community Server (GPL)
 
Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.
 
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
 
root@mysqldb 15:18: [(none)]> use hjxdb;
Database changed
root@mysqldb 15:18: [hjxdb]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| hjxdb             |
| mysql             |
| performance_schema |
| test               |
+--------------------+
5 rows in set (0.00 sec)
 
root@mysqldb 15:18: [hjxdb]> select * from user;
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
| id | username | email         | cell_phone | password | school_code | register_time       | usertype | state |
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
| 1 | apple   | apple@123.com | 12233334444 | hahaha   |         123 | 2017-09-14 13:58:03 |       1 |     2 |
| 2 | pple     | pple@123.com | 12233334445 | hahaha   |         124 | 2017-09-14 13:58:10 |       1 |     2 |
| 3 | ple     | ple@123.com   | 12233334446 | hahaha   |         125 | 2017-09-14 13:58:16 |       1 |     2 |
| 4 | le       | le@123.com   | 12233334447 | hahaha   |         126 | 2017-09-14 13:58:34 |       1 |     2 |
+----+----------+---------------+-------------+----------+-------------+---------------------+----------+-------+
4 rows in set (0.00 sec)
```