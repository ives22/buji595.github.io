---
title: ftp虚拟用户
date: 2019-04-23 22:30:00
tags: Linux运维日常
categories: Linux
copyright: true
---
### 需求：
进（账户）\
username: ftpComeSsbq\
password: ftp_come_#@UkieO9\
上传文件的目录：/opt/ftp/come

出（账户）\
username: ftpOutSsbq\
password: ftp_out_#@45oUkie\
上床文件的目录：/opt/ftp/out

### 具体步骤

1 安装软件
```
# yum -y install vsftpd
```
2 创建相应的ftp数据目录
```
# mkdir -p /opt/ftp/{come,out}
```
3 创建一个用户提供给虚拟用户使用
```
# useradd -s /sbin/nologin virtual
```
4 将ftp数据目录设置成virtual用户
```
# chown virtual. /opt/ftp/ -R
# ll /opt/ftp/
total 8
drwxr-xr-x 2 virtual virtual 4096 Apr 17 12:07 come
drwxr-xr-x 2 virtual virtual 4096 Apr 17 12:07 out
```
5 创建虚拟帐号与密码的文本文件(一行账号，一行密码, 注意不要有多余的空格)
```
# vim /etc/vsftpd/logins.txt
ftpComeSsbq
ftp_come_#@UkieO9
ftpOutSsbq
ftp_out_#@45oUkie
```
6 将创建好的密码文件txt格式转换db格式
```
# db_load -T -t hash -f /etc/vsftpd/logins.txt /etc/vsftpd/login.db
```
7 定义db文件的权限
```
# chmod 600 /etc/vsftpd/login.db
```
8 定义pam认证文件
```
# vim /etc/pam.d/ftp
auth  required  /lib64/security/pam_userdb.so  db=/etc/vsftpd/login
account  required  /lib64/security/pam_userdb.so  db=/etc/vsftpd/login
```
9 配置vsftpd主配置文件
```
# cp /etc/vsftpd/vsftpd.conf{,.bak}
# vim /etc/vsftpd/vsftpd.conf
#禁止匿名登录FTP服务器
anonymous_enable=NO
#允许本地用户登录FTP服务器
local_enable=YES
#可以上传(全局控制) 
write_enable=NO
#匿名用户可以上传
anon_upload_enable=NO
#匿名用户可以建目录
anon_mkdir_write_enable=NO
#匿名用户修改删除
anon_other_write_enable=NO
#全部用户被限制在主目录
chroot_local_user=YES
#将所有用户看成虚拟用户guest
guest_enable=YES
#指定虚拟用户，也就是将guest用户映射到virtual用户
guest_username=virtual
#指定为独立服务
listen=YES
#指定监听的端口
listen_port=21
#开启被动模式
pasv_enable=YES
#FTP服务器公网IP
pasv_address=120.79.93.66
#设置被动模式下，建立数据传输可使用port范围的最小值
pasv_min_port=10000
#设置被动模式下，建立数据传输可使用port范围的最大值
pasv_max_port=10088
#是否允许匿名用户下载全局可读的文件
anon_world_readable_only=NO
#指定虚拟用户配置文件的路径
user_config_dir=/etc/vsftpd/user_conf
```
10 创建上面配置文件中指定的子配置文件目录 user_conf
```
# mkdir /etc/vsftpd/user_conf
```
11 定义 ftpComeSsbq 用户的配置文件
```
# vim /etc/vsftpd/user_conf/ftpComeSsbq
write_enable=YES
anon_world_readable_only=no
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
local_root=/opt/ftp/come
```
12 定义 ftpOutSsbq 用户的配置文件
```
# vim /etc/vsftpd/user_conf/ftpOutSsbq
write_enable=YES
anon_world_readable_only=no
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
local_root=/opt/ftp/out
```
13 启动vsftpd
```
# service vsftpd start
```
14 测试
...
















