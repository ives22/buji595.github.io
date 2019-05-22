---
title: linux保存history到日志文件
date: 2019-05-22 16:30:09
tags: Linux运维日常
categories: Linux运维日常
copyright: true
---
>在linux下面，为了保证服务器安全，通常会记录所敲命令的历史记录，但是记录为1000条，并且退出重新登录后，之前的变会没有了，通过编辑`/etc/bashrc`文件记录历史命令到日志文件下面，并已登录来源`IP`，登录用户名，登录时间命名日志文件名字

##### 查看默认的history
![默认的history](https://upload-images.jianshu.io/upload_images/11763553-a2f73d6c0783224d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 编辑/etc/bashrc
编辑`/etc/bashrc`文件，加入以下内容，也可以放在`/etc/profile`文件里
```
# vim /etc/bashrc
USER_IP=$(echo -e "\033[31m\033[1m`who -u am i 2>/dev/null| awk '{print $NF}'|sed -e 's/[()]//g'`\033[0m")
IP=$(who -u am i 2>/dev/null| awk '{print $NF}'|sed -e 's/[()]//g')
USER=$(whoami)
USER_NAME=`echo -e "\033[36m\033[1m$(whoami) \033[0m"`
HISTFILESIZE=100000
HISTSIZE=4096
HISTTIMEFORMAT="%F %T  $USER_IP  $USER_NAME  "
if [ "$USER_IP" = "" ]
then
   USER_IP=`hostname`
fi

if [ ! -d /var/log/history ]
then
   mkdir /var/log/history
   chmod 777 /var/log/history
fi

if [ ! -d /var/log/history/${LOGNAME} ]
then
    mkdir /var/log/history/${LOGNAME}
    chmod 300 /var/log/history/${LOGNAME}
fi
DT=`date "+%Y%m%d_%H%M"`
export HISTFILE="/var/log/history/${LOGNAME}/${DT}_${USER}_${IP}"
chmod 600 /var/log/history/${LOGNAME}/* 2>/dev/null
```
退出重新登录再次查看
![](https://upload-images.jianshu.io/upload_images/11763553-dad4faed53395912.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
查看前面保留的历史记录日志
```
[root@salt-master ~]# ll /var/log/history/
总用量 0
d-wx------. 2 root root 6 5月  22 14:27 root

[root@salt-master ~]# ll /var/log/history/root/
总用量 4
-rw-------. 1 root root 123 5月  22 14:29 20190522_1427_root_192.168.1.123

[root@salt-master ~]# cat /var/log/history/root/20190522_1427_root_192.168.1.123 
#1558506461
ls
#1558506467
history
#1558506519
ll /var/log/history/
#1558506571
ll /var/log/history/root/
#1558506574
exit
```