---
title: saltstack基本入门
date: 2019-05-13 15:47:05
tags: Saltstack
categories: Saltstack
copyright: true
---

1.配置 yum 仓库
```
# 使用官方自带yum
[root@salt-master ~]# yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest.el7.noarch.rpm
# 或者使用阿里云的yum（建议使用阿里云的，速度快一点）
[root@salt-master ~]# yum -y install https://mirrors.aliyun.com/saltstack/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
[root@salt-master ~]# sed -i "s/repo.saltstack.com/mirrors.aliyun.com\/saltstack/g" /etc/yum.repos.d/salt-latest.repo
[root@salt-master ~]# yum clean all
[root@salt-master ~]# yum makecache
```
2.安装Master，并启动服务
```
[root@salt-master ~]# yum -y install salt-master
[root@salt-master ~]# systemctl enable salt-master
[root@salt-master ~]# systemctl start salt-master
```
3.安装 Salt-Minion 指向 Salt-Master 网络地址
```
[root@salt-minion01 ~]# yum -y install salt-minion
```