---
title: MySQL自动备份脚本
date: 2019-04-24 17:56:56
tags: Linux运维日常
categories: Shell
copyright: true
---

##### 通过 shell 脚本定时备份数据库，并定时清理
`ceshi yixia `

```html
#!/bin/bash
#Desc:本地数据库备份脚本
#Date:2017-12-24
#by:Lee-YJ
#运行脚本前先创建一个备份用户，并授予权限
#mysql> grant select,insert,lock tables,show view,trigger  on *.* to back@"127.0.0.1" identified by "oodaeh7phoe1iboh7Jua";
#mysql> flush privileges;

mysqluser=back                    #用于数据库备份的用户名
mysqlpassw=oodaeh7phoe1iboh7Jua   #用户密码
mysqlhost=127.0.0.1               #连接数据库的host
bakpath=/opt/data/Back/DBbak          #备份数据库存放的路径

bakdate=`date '+%Y%m%d'`
baktime=`date '+%H%M'`

#----------------------------获取mysql中的数据库名---------------------------------
/usr/bin/mysql -u$mysqluser -h $mysqlhost -p$mysqlpassw -e "show databases;"|grep -v Database|grep -v information_schema |grep -v performance_schema|grep -v mysql|grep -v test > /tmp/DBname

#----------------------------数据库备份--mysqldump备份------------------------------

if [ ! -d "$bakpath" ];then
    mkdir -p $bakpath
fi

cd $bakpath

if [ ! -d "$bakdate" ];then
    mkdir $bakdate
fi

cd $bakdate

echo "------------------------$bakdate$baktime-------------------------------------" >> $bakpath/DBbak.log
for DBname in `cat /tmp/DBname`;do
    if [ ! -d "$DBname" ];then
        mkdir $DBname
    fi
    /usr/bin/mysqldump --routines --triggers -u$mysqluser -h $mysqlhost -p$mysqlpassw $DBname  > $DBname/$baktime\.sql
    if [ $? = 0 ];then
        echo "$DBname Back OK!" >> $bakpath/DBbak.log
    else
        echo "$DBname Back ERROR!" >> $bakpath/DBbak.log
    fi
done
echo "" >> $bakpath/DBbak.log 


#----------------------------删除5天之前的数据备份，并随机保留一份--------------------------------
shopt -s extglob
olddate=`date '+%Y%m%d' -d "-6 days"`
cd $bakpath/$olddate
for deldbname in `ls $bakpath/$olddate`;do
    cd $deldbname
    file=`ls| sort -R | head -n1`
    rm -f !($file)
done

if [ ! -d "$bakpath/Oldest" ];then
    mkdir -p $bakpath/Oldest
fi

mv $bakpath/$olddate $bakpath/Oldest/

rm -f /tmp/DBname



```

