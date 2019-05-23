---
title: saltstack使用salt-ssh
date: 2019-05-20 09:43:05
tags: Saltstack
categories: Saltstack
copyright: true
---
## salt-ssh 介绍
[参考官档](https://docs.saltstack.com/en/latest/topics/ssh/index.html)
>`salt-ssh`是 `0.17.0` 新引入的一个功能，不需要minion对客户端进行管理，也可以不需要`master`；`salt-ssh`也支持`salt`大部分的功能：比如`grains`,`modules`,`state`等；`salt-ssh`没有使用`ZeroMQ`的通信架构，执行是串行模式

## salt-ssh执行原理
>- `salt-ssh`是在`salt`基础上打了一个`python`包上传到客户端的默认`tmp`目录下, 在客户端上面解压并执行返回结果,最后删除`tmp`上传的临时文件。
>- `salt-minion`方法是`salt-mater`先执行语法验证，验证通过后发送到`minion`，`minion`收到`Msater`的状态文件默认保存在`/var/cache/salt/minion`
>- `salt-ssh`和`salt-minion`可以共存，`salt-minion`不依赖于`ssh`服务


## 安装配置
1）安装`salt-ssh`
```
[root@salt-master ~]# yum -y install salt-ssh
```
2）修改`roster`文件，配置需要管理的机器
```
[root@salt-master ~]# vim /etc/salt/roster
salt-minion01:
  host: 192.168.1.31
  user: root
  passwd: 123456
  port: 22

salt-minion02:
  host: 192.168.1.32
  user: root
  passwd: 123456
  port: 22
```
3）管理测试 (说明，如果第一次需要输入`yes或no`进行`ssh`认证，可以加`-i`参数自动认证)
```
[root@salt-master ~]# salt-ssh '*' test.ping -i
salt-minion01:
    True
salt-minion02:
    True
```
4）`salt-ssh`命令参数
```
-r, –raw, –raw-shell # 直接使用shell命令
–priv #指定SSH私有密钥文件
–roster #定义使用哪个roster系统，如果定义了一个后端数据库，扫描方式，或者用户自定义的的roster系统，默认的就是/etc/salt/roster文件
–roster-file #指定roster文件
–refresh, –refresh-cache #刷新cache，如果target的grains改变会自动刷新
–max-procs #指定进程数，默认为25
-i, –ignore-host-keys #当ssh连接时，忽略keys
–passwd #指定默认密码
–key-deploy #配置keys 设置这个参数对于所有minions用来部署ssh-key认证，
 这个参和–passwd结合起来使用会使初始化部署很快很方便。当调用master模块时，并加上参数 –key-deploy 即可在minions生成keys，下次开始就不使用密码
```
5）`salt-ssh`执行命令
```
[root@salt-master ~]# salt-ssh '*' -r "uptime"
salt-minion01:
    ----------
    retcode:
        0
    stderr:
    stdout:
        root@192.168.1.31's password: 
         10:06:08 up 1 day, 17:05,  2 users,  load average: 0.00, 0.01, 0.05
salt-minion02:
    ----------
    retcode:
        0
    stderr:
    stdout:
        root@192.168.1.32's password: 
         10:06:08 up 1 day, 17:16,  2 users,  load average: 0.03, 0.06, 0.06
```
6）`salt-ssh`执行状态模块
```
[root@salt-master ~]# salt-ssh '*' state.sls modules.haproxy.service saltenv=prod
salt-minion02:
----------
          ID: haproxy-install
    Function: pkg.installed
        Name: haproxy
      Result: True
     Comment: All specified packages are already installed
     Started: 10:09:48.541019
    Duration: 10161.705 ms
     Changes:   
----------
          ID: haproxy-config
    Function: file.managed
        Name: /etc/haproxy/haproxy.cfg
      Result: True
     Comment: File /etc/haproxy/haproxy.cfg is in the correct state
     Started: 10:09:59.020659
    Duration: 54.263 ms
     Changes:   
----------
          ID: haproxy-service
    Function: service.running
        Name: haproxy
      Result: True
     Comment: The service haproxy is already running
     Started: 10:09:59.079110
    Duration: 128.052 ms
     Changes:   

Summary for salt-minion02
------------
Succeeded: 3
Failed:    0
------------
Total states run:     3
Total run time:  10.344 s
salt-minion01:
----------
          ID: haproxy-install
    Function: pkg.installed
        Name: haproxy
      Result: True
     Comment: All specified packages are already installed
     Started: 10:10:00.561015
    Duration: 2862.018 ms
     Changes:   
----------
          ID: haproxy-config
    Function: file.managed
        Name: /etc/haproxy/haproxy.cfg
      Result: True
     Comment: File /etc/haproxy/haproxy.cfg is in the correct state
     Started: 10:10:03.905713
    Duration: 220.443 ms
     Changes:   
----------
          ID: haproxy-service
    Function: service.running
        Name: haproxy
      Result: True
     Comment: The service haproxy is already running
     Started: 10:10:04.135607
    Duration: 230.231 ms
     Changes:   

Summary for salt-minion01
------------
Succeeded: 3
Failed:    0
------------
Total states run:     3
Total run time:   3.313 s
```
## Roster说明
>`salt-ssh`需要一个名单系统来确定哪些执行目标，`Salt`的`0.17.0`版本中`salt-ssh`引入`roste`r系统
`roster`系统编译成了一个数据结构，包含了`targets`，这些`targets`是一个目标系统主机列表和或如连接到这些`targets`。

配置文件说明：
```
# target的信息
    host:        # 远端主机的ip地址或者dns域名
    user:        # 登录的用户
    passwd:      # 用户密码,如果不使用此选项，则默认使用秘钥方式
# 可选的部分
    port:        #ssh端口
    sudo:        #可以通过sudo
    tty:         # 如果设置了sudo，设置这个参数为true
    priv:        # ssh秘钥的文件路径
    timeout:     # 当建立链接时等待响应时间的秒数
    minion_opts: # minion的位置路径
    thin_dir:    # target系统的存储目录，默认是/tmp/salt-<hash>
    cmd_umask:   # 使用salt-call命令的umask值
```