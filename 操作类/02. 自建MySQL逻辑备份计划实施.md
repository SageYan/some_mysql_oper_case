[TOC]

# 自建MySQL逻辑备份计划实施

### 数据库添加备份专用授权

```sql

mysql> grant select,reload,show databases,super,lock tables,replication client,show view,event,file on *.* to backup@'localhost' identified by 'abc123';
Query OK, 0 rows affected (0.00 sec)
 
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

### 备份脚本

```sql
系统库也一起备份

[root@zeji-git-server mysql]# vim backup.sh
[root@zeji-git-server mysql]# cd
[root@zeji-git-server ~]# ll /usr/local/mysql/backup.sh
-rw-r--r-- 1 root root 685 Jan  2 13:48 /usr/local/mysql/backup.sh
[root@zeji-git-server ~]# cat /usr/local/mysql/backup.sh
#!/bin/bash
DBUser=backup
DBPwd=abc123
DBName=mysql
BackupPath="/alidata/backup"
BackupFile="$DBName-"$(date +%y%m%d_%H)".sql"
# Backup Dest directory, change this if you have someother location
if !(test -d $BackupPath)
then
mkdir $BackupPath -p
fi
cd $BackupPath
mysqldump -u$DBUser -p$DBPwd -A --opt --default-character-set=utf8  --single-transaction --hex-blob --skip-triggers --max_allowed_packet=824288000 > "$BackupPath"/"$BackupFile"
#Delete sql type file
find "$BackupPath" -name "$DBname*[log,sql]" -type f -mtime +14 -exec rm -rf {} \;
```

### crontab配置

```shell
[root@zeji-git-server ~]# crontab -e
crontab: installing new crontab
 
#crontab 每日24点备份数据库
1 0 * * * /bin/bash /usr/local/mysql/backup.sh
```

