---
title: saltstack远程执行
date: 2019-05-14 17:47:05
tags: Saltstack
categories: Saltstack
copyright: true
---
安装完`Saltstack`后可以立即执行`shell`命令，更新软件包并将文件同时分不到所有受管系统。所有回复都以一致的可配置格式返回。远程执行参考文档：http://docs.saltstack.cn/topics/tutorials/modules.html
```
[root@salt-master ~]# salt '*' cmd.run "uptime"
salt-minion01:
     15:23:08 up 1 day, 58 min,  2 users,  load average: 0.00, 0.03, 0.08
salt-minion02:
     15:23:08 up 21:38,  2 users,  load average: 0.00, 0.04, 0.10
salt-minion03:
     15:23:08 up 21:36,  2 users,  load average: 0.00, 0.04, 0.10
```

## Salt命令的结构语法
```
salt '<target>' <function> [arguments]
```

![](https://upload-images.jianshu.io/upload_images/11763553-e5e69a972680590f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 目标主机Target
![](https://upload-images.jianshu.io/upload_images/11763553-e3ed49a6c6b6633f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1、通配符匹配
```
[root@salt-master ~]# salt '*' test.ping
[root@salt-master ~]# salt 'salt-minion01' test.ping
[root@salt-master ~]# salt '*01' test.ping
[root@salt-master ~]# salt 'salt-minion0[1|2]' test.ping
[root@salt-master ~]# salt 'salt-minion0[!1|2]' test.ping
[root@salt-master ~]# salt 'salt-minion0?' test.ping
```
2、列表匹配
```
[root@salt-master ~]# salt -L 'salt-minion01,salt-minion02' test.ping
```
3、正则匹配
```
[root@salt-master ~]# salt -E '^salt' test.ping
[root@salt-master ~]# salt -E '^salt.*2$' test.ping
```
4、IP匹配
```
[root@salt-master ~]# salt -S '192.168.1.32' test.ping
[root@salt-master ~]# salt -S '192.168.1.0/24' test.ping
```
5、复合匹配
```
[root@salt-master ~]# salt -C 'G@os:CentOS and S@192.168.1.32' test.ping
```
6、分组匹配
```
[root@salt-master ~]# vim /etc/salt/master
nodegroups:
  webserver: 'salt-minion01,salt-minion02'
  dbserver: 'salt-minion03
[root@salt-master ~]# systemctl restart salt-master
[root@salt-master ~]# salt -N 'webserver' test.ping
[root@salt-master ~]# salt -N 'dbserver' test.ping
```
7、Grains匹配
```
[root@salt-master ~]# salt -G 'os:CentOS' test.ping
[root@salt-master ~]# salt -G 'localhost:salt-minion02' test.ping
```
> 说明：上面这些匹配方式在`top.sls`文件中同样适用。
## 模块Module
![](https://upload-images.jianshu.io/upload_images/11763553-18184de0fb7156aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> `test`模块多用于测试
`user`模块用于用户管理
`cmd`模块可以执行任意`shell`命令
`pkg`模块用于软件包管理
`file`模块多用于配置
`service`模块用于服务管理

[所有模块列表](http://docs.saltstack.cn/ref/modules/all/index.html)

**test模块**
```
模块名：test
功能：用于测试
[root@salt-master ~]# salt '*' test.ping
```
**user模块**
```
参考：http://docs.saltstack.cn/ref/modules/all/salt.modules.useradd.html#module-salt.modules.useradd
# salt '*' user.add name <uid> <gid> <groups> <home> <shell>
[root@salt-master ~]# salt '*' user.add testuser
```
**cmd模块**
```
模块名：cmd
功能：实现远程的命令行调用执行（默认具备root操作权限，使用时需评估风险）
#查看所有minion内存和磁盘使用情况
[root@salt-master ~]# salt '*' cmd.run "free -m"
[root@salt-master ~]# salt '*' cmd.run "df -h"
```
**pkg模块**
```
模块名：pkg
功能：软件包状态管理，会根据操作系统不同，选择对应的安装方式（如CentOS系统默认使用yum，Debian系统默认使用apt-get）

#安装
[root@salt-master ~]# salt '*' pkg.install "vsftpd" 
#卸载
[root@salt-master ~]# salt '*' pkg.remove "vsftpd"
#安装最新版本
[root@salt-master ~]# salt '*' pkg.latest_version "vsftpd"
#更新软件包
[root@salt-master ~]# salt '*' pkg.upgrade "vsftpd"

#查看帮助手册
[root@salt-master ~]# salt '*' pkg
```
**file模块**
```
模块名：file
功能：被控主机常见的文件操作，包括文件读写、权限、查找、校验

#校验所有minion主机文件的加密信息，支持md5、sha1、sha224、shs256、sha384、sha512加密算法
[root@salt-master ~]# salt '*' file.get_sum /etc/passwd md5

#修改所有minion主机/etc/passwd文件的属组、用户权限、等价于chown root:root /etc/passwd
[root@salt-master ~]# salt '*' file.chown /etc/passwd root root

#获取所有minion主机/etc/passwd的stats信息
[root@salt-master ~]# salt '*' file.stats /etc/passwd

#获取所有minion主机/etc/passwd的权限mode,如755,644
[root@salt-master ~]# salt '*' file.get_mode /etc/passwd

#修改所有minion主机/etc/passwd的权限mode为0644
[root@salt-master ~]# salt '*' file.set_mode /etc/passwd 0644

#在所有minion主机创建/opt/test目录
[root@salt-master ~]# salt '*' file.mkdir /opt/test

#在所有minion主机穿件/tmp/test.conf文件
[root@salt-master ~]# salt '*' file.touch /tmp/test.conf

#将所有minion主机/tmp/test.conf文件追加内容'maxclient 100'
[root@salt-master ~]# salt '*' file.append /tmp/test.conf 'maxclient 100'

#删除所有minion主机的/tmp/test.conf文件
[root@salt-master ~]# salt '*' file.remove /tmp/test.conf
```
**service模块**
```
模块名：service
功能：被控主机程序包服务管理

#开启（enable）禁用（disable）
salt '*' service.enable <service name>
salt '*' service.disabled <service name>

#reload、restart、start、stop、status操作
salt '*' service.reload <service name>
salt '*' service.restart <service name>
salt '*' service.start <service name>
salt '*' service.stop <service name>
salt '*' service.status <service name>
```





