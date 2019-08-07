[TOC]

# 处理CPU飙高，杀死会话脚本



## kill_long_session.sh

```shell
`#!/bin/bash``# kill掉 指定数据库kyh中超过1分钟的会话id，暴力，用于CPU持续飙高，临时解决问题``user=booboo``password=``host=``""``port=3306` `# processlist snapshot``mysql -u$user -p$password -h$host  -P$port -e ``"select concat('kill ',id,';'),'KYHINFO',now(),id,user,host,db,command,time,state,info from information_schema.processlist where db='kyh' and  time > 60"`  `> tmpfile` `# save``cat` `tmpfile >> processlist_history.log` `# kill``awk` `-F ``'KYHINFO'` `'{if (NR != 1) print $1 }'` `tmpfile | mysql -u$user -p$password -h$host  -P$port`
```



## kill_all_session



```shell
`#!/bin/bash``# kill掉 数据库中所有的会话id，暴力，用于CPU持续飙高，临时解决问题``user=root``password=``"(Uploo00king)"``host=localhost``port=3306`  `mysql -u$user -p$password -h$host  -P$port -e ``"select concat('kill ',id,';') from information_schema.processlist"` `> tmpfile` `awk` `'{if (NR != 1) print $0 }'` `tmpfile | mysql -u$user -p$password -h$host  -P$port`
```

## kill_databases_session



```shell
`#!/bin/bash``# kill掉 指定数据库中所有的会话id，暴力，用于CPU持续飙高，临时解决问题``user=root``password=``"(Uploo00king)"``host=localhost``port=3306`  `mysql -u$user -p$password -h$host  -P$port -e ``"select concat('kill ',id,';') from information_schema.processlist where db in ("``dbname``","``dbname2") > tmpfile` `awk` `'{if (NR != 1) print $0 }'` `tmpfile | mysql -u$user -p$password -h$host  -P$port`
```

## kille_blocking_trx_id



```shell
`#!/bin/bash``# kill掉 ecshoptest库中的导致lock wait会话id``user=root``password=``"(Uploo00king)"``host=localhost``port=3306`  `mysql -u$user -p$password -h$host  -P$port -e ``"select concat('kill ',id,';') from information_schema.processlist,information_schema.innodb_trx  where trx_mysql_thread_id=id and trx_id in (select blocking_trx_id from (select blocking_trx_id, count(blocking_trx_id) as countnum from (select a.trx_id,a.trx_state,b.requesting_trx_id,b.blocking_trx_id from information_schema.innodb_lock_waits as  b left join information_schema.innodb_trx as a on a.trx_id=b.requesting_trx_id) as t1 group by blocking_trx_id order by  countnum desc limit 1) c) ;"` `> tmpfile` `awk` `'{if (NR != 1) print $0 }'` `tmpfile | mysql -u$user -p$password -h$host  -P$port`
```

## forget_password



```shell
`#!/bin/bash``# mysql的root密码不停服务破解方式` `user=booboo``password=``"(Uploo00king)"``host=localhost``port=3306``datadir=``"/alidata/mysql/data"``database=``"cloudcare"``dbdir=${datadir}/${database}``echo` `${dbdir}``rootpwd=``'(Uploo00king)'`   `# linux系统内的操作，拷贝mysql下的用户表，到cloudcare库下``\``cp` `-avx ${datadir}``/mysql/user``.* ${dbdir}` `# 登陆低权限用户连接mysql,修改root密码``echo` `"update ${database}.user set authentication_string=password('${rootpwd}') where user='root';"` `| mysql -u$user -p$password -h$host  -P$port $database` `# 拷贝更改后的user表到mysql库下``\``cp` `-avx ${dbdir}``/user``.*  ${datadir}``/mysql/` `# 查找mysqld的父进程``mysqld_pid=`pgrep -n mysqld`` `# 向mysqld发送SIGHUP信号，强制刷新``kill` `-SIGHUP ${mysqld_pid}` `# ok 牢记密码即可``echo` `"root password is $rootpwd , Please remember the password!!! "`
```