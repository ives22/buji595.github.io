---
title: saltstack基本入门
date: 2019-05-13 15:47:05
tags: Saltstack
categories: Saltstack
copyright: true
---
### saltstack介绍
 >Salt，,一种全新的基础设施管理方式，部署轻松，在几分钟内可运行起来，扩展性好，很容易管理上万台服务器，速度够快，服务器之间秒级通讯

**主要功能**
远程执行
配置管理

[Stalstack官方文档](http://docs.saltstack.cn/)
### 1 快速安装
1.1 配置 yum 仓库
```
# 使用官方自带yum
[root@salt-master ~]# yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest.el7.noarch.rpm
# 或者使用阿里云的yum（建议使用阿里云的，速度快一点）
[root@salt-master ~]# yum -y install https://mirrors.aliyun.com/saltstack/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
[root@salt-master ~]# sed -i "s/repo.saltstack.com/mirrors.aliyun.com\/saltstack/g" /etc/yum.repos.d/salt-latest.repo
[root@salt-master ~]# yum clean all
[root@salt-master ~]# yum makecache
```
1.2 安装Master，并启动服务
```
[root@salt-master ~]# yum -y install salt-master
[root@salt-master ~]# systemctl enable salt-master
[root@salt-master ~]# systemctl start salt-master
```
1.3 安装 Salt-Minion 指向 Salt-Master 网络地址
```
[root@salt-minion01 ~]# yum -y install salt-minion
# 可以使用主机名，也可以使用IP地址
[root@salt-minion01 ~]# cp /etc/salt/minion{,.back}
[root@salt-minion01 ~]# sed -i '/#master: /c\master: salt-master' /etc/salt/minion
[root@salt-minion01 ~]# systemctl enable salt-minion
[root@salt-minion01 ~]# systemctl start salt-minion
```
### 2 SaltStack认证方式
> `Salt` 的数据传输是通过 `AES` 加密，`Master` 和 `Minion` 之前在通信之前，需要进行认证。
`Salt` 通过认证的方式保证安全性，完成一次认证后，Master 就可以控制 Minion 来完成各项工作了。

>1.  `minion` 在第一次启动时候，会在 `/etc/salt/pki/minion/` 下自动生成 `minion.pem(private key)` 和 `minion.pub(public key) `, 然后将 `minion.pub` 发送给 `master `
>2. `master` 在第一次启动时，会在 `/etc/salt/pki/master/` 下自动生成 `master.pem` 和 `master.pub` ；并且会接收到 `minion` 的 `public key` , 通过 `salt-key` 命令接收 `minion public key`， 会在 `master` 的 `/etc/salt/pki/master/minions` 目录下存放以 `minion id` 命令的 `public key` ；验证成功后同时 `minion` 会保存一份 `master public key` 在 minion的 `/etc/salt/pki/minion/minion_master.pub`里。

**Salt认证原理总结**
> `minion`将自己的公钥发送给`master`
`master`认证后再将自己的公钥也发送给`minion`端

**Master端认证示例**
根据上面提到的认证原理，先看下未认证前的`master`和`minion`的`pki`目录
```
# master上查看
[root@salt-master ~]# tree /etc/salt/pki/
/etc/salt/pki/
├── master
│   ├── master.pem
│   ├── master.pub
│   ├── minions
│   ├── minions_autosign
│   ├── minions_denied
│   ├── minions_pre
│   │   └── salt-minion01
│   └── minions_rejected
└── minion

# minion上查看
[root@salt-minion01 ~]# tree /etc/salt/pki/
/etc/salt/pki/
├── master
└── minion
    ├── minion.pem
    └── minion.pub
```
`salt-key`命令解释：
```
[root@salt-master ~]# salt-key -L 
Accepted Keys:        #已经接受的key
Denied Keys:          #拒绝的key
Unaccepted Keys:      #未加入的key
Rejected Keys:        #吊销的key

#常用参数
-L  #查看KEY状态
-A  #允许所有
-D  #删除所有
-a  #认证指定的key
-d  #删除指定的key
-r  #注销掉指定key（该状态为未被认证）

#配置master自动接受请求认证(master上配置 /etc/salt/master)
auto_accept: True
```
`salt-key`认证
```
#列出当前所有的key
[root@salt-master ~]# salt-key -L 
Accepted Keys:
Denied Keys:
Unaccepted Keys:
salt-minion01
Rejected Keys:

#添加指定minion的key
[root@salt-master ~]# salt-key -a salt-minion01 -y
The following keys are going to be accepted:
Unaccepted Keys:
salt-minion01
Key for minion salt-minion01 accepted.
#添加所有minion的key
[root@salt-master ~]# salt-key -A -y

[root@salt-master ~]# salt-key -L 
Accepted Keys:
salt-minion01
Denied Keys:
Unaccepted Keys:
Rejected Keys:
```
上面认证完成后再次查看`master`和`minion`的`pki`目录
```
#master上
[root@salt-master ~]# tree /etc/salt/pki/
/etc/salt/pki/
├── master
│   ├── master.pem
│   ├── master.pub
│   ├── minions
│   │   └── salt-minion01
│   ├── minions_autosign
│   ├── minions_denied
│   ├── minions_pre
│   └── minions_rejected
└── minion

#minion上
[root@salt-minion01 ~]# tree /etc/salt/pki/
/etc/salt/pki/
├── master
└── minion
    ├── minion_master.pub
    ├── minion.pem
    └── minion.pub
```
### Saltstack远程执行
>远程执行是 `Saltstack` 的核心功能之一。主要使用 `salt` 模块批量给选定的 `minion` 端执行相应的命令，并获得返回结果。

1、判断 `salt` 的 `minion` 主机是否存活
```
[root@salt-master ~]# salt '*' test.ping
salt-minion02:
    True
salt-minion03:
    True
salt-minion01:
    True

# salt saltstack自带的一个命令
# * 表示目标主机，这里表示所有目标主机
# test.ping test是saltstack中的一个模块，ping则是这个模块下面的一个方法
```
2、`saltstack`使用 `cmd.run`模块远程执行shell命令
```
#在指定目标minion节点运行uptime命令
[root@salt-master ~]# salt 'salt-minion02' cmd.run 'uptime'
salt-minion02:
     18:13:08 up 28 min,  2 users,  load average: 0.00, 0.04, 0.13
```
### Saltstack配置管理



