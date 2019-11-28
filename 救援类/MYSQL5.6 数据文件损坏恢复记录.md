[toc]

# MYSQL5.6 数据文件损坏恢复记录

## 找回损坏表的元数据（表结构数据）

* 原因：

  因为t_stan_stif是分区表，相对于历史表结构发生了改变，而且表无法打开查看表结构

* 操作方法：

   [https://github.com/BoobooWei/DBA_Mysql/blob/master/Tec/mysql_5.7_%E9%80%9A%E8%BF%87frm%E5%92%8Cibd%E5%BC%BA%E5%88%B6%E6%81%A2%E5%A4%8D%E7%BB%93%E6%9E%84%E5%92%8C%E6%95%B0%E6%8D%AE.md](https://github.com/BoobooWei/DBA_Mysql/blob/master/Tec/mysql_5.7_通过frm和ibd强制恢复结构和数据.md) 

## 利用discard、import  tablespace  恢复

1 新建5.7的数据库实例（5.6不支持分区表的discard import操作）

2 拷贝源数据库数据文件到目标端

3 在目标环境（5.7）新建数据库和表结构，尝试discard import恢复

```sql
1 导出原来数据库的表结构
mysqldump -P 3306 -uroot -proot zeusdb --no-data  >zeusdb.ddl.sql
mysqldump -P 3306 -uroot -proot activiti --no-data  >activiti.ddl.sql
mysqldump -P 3306 -uroot -proot lts --no-data  >lts.ddl.sql

2 利用元数据在恢复环境创建相关的表结构

3 discard各个数据库innodb引擎的表
例：
alter table load_address        discard tablespace;
alter table load_bact           discard tablespace;
alter table load_cert           discard tablespace;
alter table load_org_copy       discard tablespace;
alter table load_org            discard tablespace;
alter table load_pact           discard tablespace;
alter table load_person         discard tablespace;
alter table load_relation       discard tablespace;
alter table load_stif_copy      discard tablespace;
alter table load_stif           discard tablespace;
alter table load_tel            discard tablespace;

4 拷贝源数据的ibd文件到相应的数据目录
5 import相关的表空间
alter table load_address        import tablespace;
alter table load_bact           import tablespace;
alter table load_cert           import tablespace;
alter table load_org_copy       import tablespace;
alter table load_org            import tablespace;
alter table load_pact           import tablespace;
alter table load_person         import tablespace;
alter table load_relation       import tablespace;
alter table load_stif_copy      import tablespace;
alter table load_stif           import tablespace;
alter table load_tel            import tablespace;
注：
在执行到zeusdb数据库t_stan_stif分区表时，数据库实例宕机。
报错：表空间超出边界。推测部分ibd数据文件损坏。
```

## 利用undrop-for-innodb-develop工具进行恢复

简介 undrop-for-innodb 是针对 innodb 的一套数据恢复工具，可以从文件级别恢复诸如：

DROP/TRUNCATE table, 删除表中某些记录，innodb 文件被删除，文件系统损坏，磁盘 corruption 等几种情况。 

### 1 下载安装

```shell
git clone https://github.com/twindb/undrop-for-innodb.git 
unzip undrop-for-innodb-develop.zip 
yum install make gcc flex bison
make
```

### 2 利用ibdata1恢复损坏的表结构

```shell
[root@fxq2 undrop-for-innodb-develop]# ./stream_parser -f /bk/back/ibdata1 

./c_parser  -5f  pages-ibdata1/FIL_PAGE_INDEX/0000000000000001.page -t  dictionary/SYS_TABLES.sql >./dumps/default/SYS_TABLES  2>./dumps/default/SYS_TABLES.sql
./c_parser  -5f  pages-ibdata1/FIL_PAGE_INDEX/0000000000000003.page  -t  dictionary/SYS_INDEXES.sql >./dumps/default/SYS_INDEXES  2>./dumps/default/SYS_INDEXES.sql
./c_parser  -5f  pages-ibdata1/FIL_PAGE_INDEX/0000000000000002.page  -t  dictionary/SYS_COLUMNS.sql >./dumps/default/SYS_COLUMNS  2>./dumps/default/SYS_COLUMNS.sql
./c_parser  -5f  pages-ibdata1/FIL_PAGE_INDEX/0000000000000004.page  -t  dictionary/SYS_FIELDS.sql >./dumps/default/SYS_FIELDS  2>./dumps/default/SYS_FIELDS.sql

#现在我们看到的就是恢复出来的元数据记录，每张字典表有两个文件，和表名同名的文件已文本形式保存行记录，另一个SQL文件是根据表结构生成的loda data语句。
#接下来在本地库上将字典数据恢复出来，注意这里需要是另外一个mysql实例，因为为了避免数据文件被复写，我们此前已经将事故实例停止了服务。

#根据工具dictionary目录中的SQL脚本创建字典表
mysql> create database dictionary;
mysql> use dictionary;

mysql> source  ../dictionary/SYS_TABLES.sql
mysql> source  ../dictionary/SYS_INDEXES.sql
mysql> source  ../dictionary/SYS_FIELDS.sql
mysql> source  ../dictionary/SYS_COLUMNS.sql

#执行前面生成的LOAD DATA语句导入恢复的记录
shell> mysql -h127.0.0.1 -uroot  -p dictionary <SYS_TABLES.sql
shell> mysql -h127.0.0.1 -uroot  -p dictionary <SYS_COLUMNS.sql
shell> mysql -h127.0.0.1 -uroot  -p dictionary <SYS_INDEXES.sql
shell> mysql -h127.0.0.1 -uroot  -p dictionary <SYS_FIELDS.sql

#到这里工作已经基本完成，因为恢复出来的元数据信息，我们已经可以在本地实例的表中查看了。接下来就是最后一步，使用sys_parser读取表中的元数据记录，并生成DDL语句。
gcc `/usr/local/mysql/bin/mysql_config --cflags` `/usr/local/mysql/bin/mysql_config --libs` -o sys_parser sys_parser.c
#注意：最好联合mysql的安装路径进行编译，如果已经编译完成并出现如下报错的话，可以修改ld配置解决

'''[root@mydocker-test1 undrop-for-innodb]# ./sys_parser -h127.0.0.1  -uroot  -p****   -d dictionary  test_1/t1
./sys_parser: error while loading shared libraries: libmysqlclient.so.20: cannot open shared object file: No such file or directory
[root@mydocker-test1 undrop-for-innodb]# vi /etc/ld.so.conf.d/mariadb-x86_64.conf 
/opt/mysql/lib
[root@mydocker-test1 undrop-for-innodb]# ldconfig '''

执行sys_parser，需要注意的是，用作表结构恢复的实例端口必须为3306，这里没有提供端口选项
[root@mydocker-test1 undrop-for-innodb]# ./sys_parser -h127.0.0.1  -uroot -p***  -d dictionary  test_1/t1
CREATE TABLE `t1`(
    `id` INT NOT NULL,
    `name` VARCHAR(20) CHARACTER SET 'utf8' COLLATE 'utf8_general_ci',
    `age` INT,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;


#对比前面环境准备中原始的表结构，这里少了一些信息，就是二级索引。undrop-for-innodb的表结构恢复功能，只会恢复表的主干信息，不包含自增、二级索引以及外键等信息。但是这些信息我们可以通过字典数据获取。

#这里我们首先通过SYS_INDEXES表找到该表的索引信息，发现除了primary key之外还有一个二级索引name

mysql> select * from SYS_INDEXES where TABLE_ID=41;
+----------+----+---------+----------+------+-------+------------+
| TABLE_ID | ID | NAME    | N_FIELDS | TYPE | SPACE | PAGE_NO    |
+----------+----+---------+----------+------+-------+------------+
|       41 | 42 | PRIMARY |        1 |    3 |    26 | 4294967295 |
|       41 | 43 | name    |        1 |    0 |    26 | 4294967295 |
+----------+----+---------+----------+------+-------+------------+
2 rows in set (0.00 sec)

#然后通过SYS_FIELDS表查找这个索引的字段定义。

mysql> select * from SYS_FIELDS  where index_id=43;
+----------+-----+----------+
| INDEX_ID | POS | COL_NAME |
+----------+-----+----------+
|       43 |   0 | name     |
+----------+-----+----------+
1 row in set (0.00 sec)

然后我们就可以通过alter table语句恢复二级索引了。索引名为'name'，并且定义中只有name这一个字段

mysql> alter table t1 add key `name`(name);

```



#### 2.1 另外可以 通过dbsake恢复表结构

```shell
#使用方法
$ curl -s get.dbsake.net > dbsake
$ chmod u+x dbsake
$ ./dbsake frmdump [frm-file-path]

chmod u+x dbsake 
./dbsake frmdump /bk/back/zeusdb/t_stan_stif.frm 
./dbsake frmdump /bk/back/zeusdb/t_stan_stif.frm > t_stan_stif.ddl
more t_stan_stif.ddl

--
-- Table structure for table `t_stan_stif`
-- Created with MySQL Version 5.6.44
--

CREATE TABLE `t_stan_stif` (
  `id` varchar(32) NOT NULL,
  `ctif_tp` char(1) DEFAULT NULL,
  `client_tp` char(1) DEFAULT NULL,
  `ctif_id` varchar(32) DEFAULT NULL,
  `ctnm` varchar(128) DEFAULT NULL,
  `smid` varchar(128) DEFAULT NULL,
  `citp` varchar(2) DEFAULT NULL,
... ...
```

### 3 从损坏的ibd文件里解析数据

参考：

https://www.parnassusdata.com/ja/node/324

#### 3.1 解析t_stan_stif所有的ibd文件

```shell
#由于分区数量众多，使用脚本处理
#!/bin/bash
cd /mysql/rec_pages
for file in `ls /mysql/backup/zeusdb/t_stan_stif`
        do 
                 /mysql/undrop-for-innodb-develop/stream_parser -f /mysql/backup/zeusdb/t_stan_stif/${file}
        done 
        
说明：
-f 指定需要解析的文件
```

#### 3.2 处理解析完成的数据pages

```shell
#!/bin/bash
#cd /mysql/rec_pages
cd /mysql/undrop-for-innodb-develop
for a in `ls /mysql/rec_pages`
        do
                #echo $a
                for b in `ls /mysql/rec_pages/${a}/FIL_PAGE_INDEX`
                        do
                        ./c_parser -5f /mysql/rec_pages/${a}/FIL_PAGE_INDEX/${b} -t /root/cloudcare_dba/zeusdb.t_stan_stif.sql 2&> /mysql/t_stan_sql/20191117_${b}
                        #echo $b
                        #break
                        done
        #break
        done
        
#脚本说明：
1 /mysql/rec_pages pages的存放目录
2 -5f 最前面的数字4/5表示行格式REDUNDANT/COMPACT f后面指定文件
  -4Df D表示获取被删除的记录
3 -t  指定建表语句，生成的结果会根据表结构进行组织

```

#### 3.3 加载数据

```sql
--根据处理完成的数据页，生产数据load脚本
cd /mysql/t_stan_sql/

cat load_data.sql
LOAD DATA LOCAL INFILE '20191117_0000000000001057.page' REPLACE INTO TABLE  t_stan_stif  CHARACTER SET UTF8 FIELDS TERMINATED BY '\t' OPTIONALLY ENCLOSED BY '"' LINES STARTING BY 't_stan_stif\t' ( id ,  tstm ,  ctif_tp ,  client_tp ,  ctif_id ,  ctnm ,  smid ,  citp ,  citp_nt ,  ctid ,  cbat ,  cbac ,  cabm ,  ctat ,  ctac ,  cpin ,  cpba ,  cpbn ,  ctip ,  crat ,  cttp ,  tsdr ,  crpp ,  crtp ,  tcif_id ,  tcnm ,  tsmi ,  tcit ,  tcit_nt ,  tcid ,  tcat ,  tcba ,  tcbn ,  tctt ,  tcta ,  tcpn ,  tcpa ,  tpbn ,  tcip ,  tmnm ,  bptc ,  pmtc ,  ticd ,  redt ,  busi_type ,  is_succ ,  is_rept ,  extend1 ,  extend2 ,  extend3 ,  extend4 ,  extend5 ,  trans_type ,  pos_dev_id ,  trans_stat ,  bank_stat ,  gate_id ,  mer_prov ,  mer_area ,  pos_prov ,  pos_area ,  acct_date ,  ord_id ,  qgj_mer_id ,  mer_unit ,  run_dt );
... ...

source load_data.sql
--注意：
cd /mysql/undrop-for-innodb-develop		
mkdir -p dumps/default
./c_parser -5f pages-ibdata1/FIL_PAGE_INDEX/0000000000000376.page -t sakila/actor.sql > dumps/default/actor 2> dumps/default/actor_load.sql
--要简化加载，c_parser 输出LOAD DATA INFILE 命令到标准错误输出
--我们使用该文件的默认位置：dump/default
```

