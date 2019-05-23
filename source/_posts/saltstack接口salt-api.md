---
title: saltstack接口salt-api
date: 2019-05-20 22:04:05
tags: Saltstack
categories: Saltstack
copyright: true
---
## 介绍
[参考官档](https://docs.saltstack.com/en/latest/ref/netapi/all/salt.netapi.rest_cherrypy.html)
[参考官档](https://www.unixhot.com/docs/saltstack/ref/netapi/all/salt.netapi.rest_cherrypy.html#a-rest-api-for-salt)
>`SaltStack`官方提供有`REST API`格式的`salt-api`项目，将使`salt`与第三方系统集成变得更加简单。

## salt-api安装配置
1）在`salt-master`上进行安装
```
[root@salt-master ~]# yum -y install salt-api
```
2）自签名证书，生产环境可以购买（说明：如果没有`salt-call`命令，装上`salt-minion`即可，依赖于该包）
```
[root@salt-master ~]# salt-call --local tls.create_self_signed_cert
local:
    Created Private Key: "/etc/pki/tls/certs/localhost.key." Created Certificate: "/etc/pki/tls/certs/localhost.crt."
```
3）打开`include`加载子配置文件，方便管理
```
[root@salt-master ~]# vim /etc/salt/master
default_include: master.d/*.conf
```
4）配置`api`配置文件，将上面生成的证书写到配置文件
```
[root@salt-master ~]# vim /etc/salt/master.d/api.conf
rest_cherrypy:
  host: 192.168.1.30
  port: 8000
  ssl_crt: /etc/pki/tls/certs/localhost.crt
  ssl_key: /etc/pki/tls/certs/localhost.key
```
5）创建认证用户，并设置密码
```
[root@salt-master ~]# useradd -M -s /sbin/nologin saltapi
[root@salt-master ~]# echo 'saltapi' | passwd --stdin saltapi
```
6）创建认证配置文件
```
[root@salt-master ~]# vim /etc/salt/master.d/auth.conf
external_auth:
  pam:
    saltapi:
      - .*
      - '@wheel'
      - '@runner'
      - '@jobs'
```
7）重启`salt-master`和启动`salt-api`
```
[root@salt-master ~]# systemctl restart salt-master
[root@salt-master ~]# systemctl start salt-api
```
8）查看`salt-api`监听端口
```
[root@salt-master ~]# netstat -anlutp |grep 8000
tcp        0      0 192.168.1.30:8000       0.0.0.0:*               LISTEN      10904/python        
tcp        0      0 192.168.1.30:53414      192.168.1.30:8000       TIME_WAIT   - 
```
9）验证`login`登录，获取`token`字符串
```
[root@salt-master ~]# curl -sSk https://192.168.1.30:8000/login \
>     -H 'Accept: application/x-yaml' \
>     -d username=saltapi \
>     -d password=saltapi \
>     -d eauth=pam
return:
- eauth: pam
  expire: 1558663247.869537
  perms:
  - .*
  - '@wheel'
  - '@runner'
  - '@jobs'
  start: 1558620047.869536
  token: e8330f642a3addd853c723d63844d29a12de9484
  user: saltapi
```
10）通过`api`执行`test.ping`测试连通性
```
[root@salt-master ~]# curl -sSk https://192.168.1.30:8000 \
>     -H 'Accept: application/x-yaml' \
>     -H 'X-Auth-Token: e8330f642a3addd853c723d63844d29a12de9484'\
>     -d client=local \
>     -d tgt='*' \
>     -d fun=test.ping
return:
- salt-minion01: true
  salt-minion02: true
  salt-minion03: true
  salt-minion04: true
```
11）通过`api`执行`cmd.run`
```
[root@salt-master ~]# curl -sSk https://192.168.1.30:8000 \
>     -H 'Accept: application/x-yaml' \
>     -H 'X-Auth-Token: e8330f642a3addd853c723d63844d29a12de9484'\
>     -d client=local \
>     -d tgt='*' \
>     -d fun='cmd.run' -d arg='uptime'
return:
- salt-minion01: ' 22:10:25 up 46 min,  1 user,  load average: 0.00, 0.01, 0.05'
  salt-minion02: ' 22:10:25 up 7 min,  0 users,  load average: 0.00, 0.18, 0.15'
  salt-minion03: ' 22:10:25 up 7 min,  0 users,  load average: 0.06, 0.33, 0.26'
  salt-minion04: ' 22:10:25 up 7 min,  0 users,  load average: 0.01, 0.21, 0.16'
```
12）通过`api`获取`grains`信息`
```
[root@salt-master ~]# curl -sSk https://192.168.1.30:8000/minions/salt-minion01 \
>     -H 'Accept: application/x-yaml' \
>     -H 'X-Auth-Token: e8330f642a3addd853c723d63844d29a12de9484'
return:
- salt-minion01:
    SSDs: []
    biosreleasedate: 05/19/2017
    biosversion: '6.00'
    cpu_flags:
    - fpu
    - vme
    - de
    - pse
    - tsc
.....
```
13）使用`json`格式
```
[root@salt-master ~]# curl -sSk https://192.168.1.30:8000/minions/salt-minion01 \
>     -H 'Accept: application/json' \
>     -H 'X-Auth-Token: e8330f642a3addd853c723d63844d29a12de9484'
{"return": [{"salt-minion01": {"biosversion": "6.00", "kernel": "Linux", "domain": "", "uid": 0, "zmqversion": "4.1.4", "kernelrelease": "3.10.0-693.el7.x86_64", "selinux": {"enforced": "Disabled", "enabled": false}, "serialnumber": "VMware-56 4d 9e a0 21 56 90 87-cd 89 69 32 13 94 17 44", "pid": 1449, "fqdns": [], "ip_interfaces": {"lo": ["127.0.0.1", "::1"], "virbr0": ["192.168.122.1"], "virbr0-nic": [], "ens33": ["192.168.1.31", "192.168.1.100", "fe80::20c:29ff:fe94:1744"]}, "groupname": "root", "fqdn_ip6": ["fe80::20c:29ff:fe94:1744"],
.......
```

## 总结
>`salt-api`必须使用`https`，生产环境建议使用可信证书
>当`salt-api`服务重启后原`token`失效

