---
title: saltstack数据系统
date: 2019-05-17 17:31:05
tags: Saltstack
categories: Saltstack
copyright: true
---
## 数据系统Grains
>1、`Grains`是`SaltStack`收集的有关底层管理系统的静态信息。包括操作系统版本、域名、IP地址、内存、内核、CPU、操作系统类型以及许多其他系统属性。`Minion` 收集的信息，可以作为`Master`端匹配目标。
>2、如果需要自定义`grains`，需要添加到`Salt Minion`的`/etc/salt/grains`文件中（配置文件中定义的默认路径），也可以直接写在配置文件`/etc/salt/minion`中

[Grains官方文档](https://docs.saltstack.com/en/latest/topics/grains/)
1）资产管理，信息查询
```
#列出所有可用的grains状态模块
[root@salt-master ~]# salt '*' grains.ls
#打印所有状态信息
[root@salt-master ~]# salt '*' grains.items
#列出每台minion的本地IP地址
[root@salt-master ~]# salt '*' grains.item fqdn_ip4
#列出每台minion的操作系统
[root@salt-master ~]# salt '*' grains.item os
```
2）用于匹配
```
[root@salt-master ~]# salt -G 'os:CentOS' test.ping
[root@salt-master ~]# salt -G 'localhost:salt-minion01' test.ping
```
3）`minion`自定义`grains`
```
#1.修改配置文件，自定义grains
[root@salt-minion01 ~]# vim /etc/salt/minion
grains:
  roles:
    - webserver
    - memcache
  ipaddr:
    - 192.168.1.32

#2.重启minion
[root@salt-minion01 ~]# systemctl restart salt-minion

#3.master上测试
[root@salt-master ~]# salt -G 'ipaddr:192.168.1.32' test.ping 
salt-minion01:
    True
```
4）`Grains`优先级问题
```
1、Grains默认核心信息
2、自定义写在/etc/salt/grains文件中的
3、自定义写在/etc/salt/minion文件中的
```
## 数据系统Pillar
>`Pillar`是动态的，`Pillar`存储在`master`上，提供给`minion`。
>`Pillar`主要记录一些加密信息，可以确保这些敏感数据不被其他`minion`看到。比如：软件版本号、用户名密码等。存储格式都是`YAML`格式

1）在`Master`端定义`Pillar`
```
[root@salt-master ~]# vim /etc/salt/master
pillar_roots:
  base:
    - /srv/pillar

[root@salt-master ~]# mkdir /srv/pillar
[root@salt-master ~]# cat /srv/pillar/zabbix.sls 
Zabbix_Server: 192.168.1.11
Zabbix_Name: zabbix.examp.com
```
2）编写`TopFile`指定`Minion`端可以使用
```
[root@salt-master ~]# cat /srv/pillar/top.sls 
base:
  'salt-minion01':
    - zabbix
```
3）刷新`Pillar`
```
[root@salt-master ~]# salt '*' saltutil.refresh_pillar
```
4）获取对应`pillar`值
```
[root@salt-master ~]# salt '*' pillar.items
salt-minion01:
    ----------
    Zabbix_Name:
        zabbix.examp.com
    Zabbix_Server:
        192.168.1.11
salt-minion03:
    ----------
salt-minion02:
    ----------

#获取指定的key
[root@salt-master ~]# salt 'salt-minion01' pillar.item Zabbix_Server
salt-minion01:
    ----------
    Zabbix_Server:
        192.168.1.11
```
>说明：如果`Master`更新了新的数值，需要刷新`Pillar`到`Minion`才可以获取

>`Pirrar`与`Grains`对比
```
类型     数据采集方式   应用场景                   定义位置
Grains   静态         minion启动时收集  数据查询  目标选择  配置管理   minion
Pillar   动态         master进行自定义  目标选择  配置管理  敏感数据   master
```