---
title: saltstack配置管理
date: 2019-05-13 15:47:05
tags: Saltstack
categories: Saltstack
copyright: true
---
## saltstack状态模块
>远程执行模块的执行是过程式，而状态是对`minion`的一种描述和定义，管理人员不需要关系部署任务如何完成的，只需要描述`minion`的状态描述。
它的和兴是写`sls(Salt State file)`文件，`sls`文件默认格式为`YAML`格式，并默认使用`jinja`模板，`jinja`是根据`django`的模板语言发展而来的语言，简单并强大，支持`for if `等循环语句。`salt state`主要用来描述系统，服务，配置文件的状态，常常被称为配置管理。

```
mysql-install:   #ID声明，必须唯一
  pkg.installed:   #state状态声明
    - pkgs:   #选项声明
      - mariadb:   #选项列表
      - mariadb-server

说明：
一个ID只能出现一次
一个ID下相同模块只能使用一次
一个ID下不可以使用多个不同模块
```
模块帮助手册
```
#列出所有状态模块
[root@salt-master ~]# salt '*' sys.list_modules
#查看指定模块的所有方法
[root@salt-master ~]# salt '*' sys.list_state_functions pkg
#查看指定模块的使用方法
[root@salt-master ~]# salt '*' sys.state_doc pkg
#查看指定模块的指定方法的用法
[root@salt-master ~]# salt '*' sys.state_doc pkg.installed
```
#### pkg软件模块
[pkg模块官档](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.pkg.html)
`pkg.installed` 软件安装
```
php-install:
  pkg.installed:
    - pkgs:
      - php
      - php-mysql: ">=5.4.16"    #指定安装版本
      - php-pdo
      - php-cli
```
指定安装最新版本的软件
```
php-install:
  pkg.latest:
    - pkgs:
      - php
      - php-mysql
      - php-pdo
      - php-cli
```
#### file文件模块
[file模块官档](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.file.html)
`file.managed` 下发文件，确保文件存在
```
[root@salt-master ~]# mkdir /srv/salt/base/files
[root@salt-master ~]# cp /etc/httpd/conf/httpd.conf /srv/salt/base/files/
[root@salt-master ~]# cat /srv/salt/base/apache_conf.sls 
apache-config:
  file.managed:
    - name: /etc/httpd/conf/httpd.conf
    - source: salt://files/httpd.conf
    - user: root
    - group: root
    - mode: 644

说明：
- name: 表示存放目的地址
- source: 表示这个文件来自哪里（说明这个文件得提前准备）

或者这样写，直接已目的地址命令ID，这样ID也表示目的地址
[root@salt-master ~]# cat /srv/salt/base/apache_conf.sls 
/etc/httpd/conf/httpd.conf:
  file.managed:
    - source: salt://files/httpd.conf
    - user: root
    - group: root
    - mode: 644
```
```
小示例：
[root@salt-master ~]# cat /srv/salt/base/test.sls 
/tmp/passwd_back:
  file.managed:
    - source: salt://files/passwd
    - user: root
    - group: root
    - mode: 644
[root@salt-master ~]# cp /etc/passwd /srv/salt/base/files/
[root@salt-master ~]# salt '*' state.sls test
[root@salt-master ~]# salt '*' cmd.run "ls -l /tmp/passwd_back"
salt-minion03:
    -rw-r--r-- 1 root root 2098 May 15 16:21 /tmp/passwd_back
salt-minion02:
    -rw-r--r-- 1 root root 2098 May 15 16:21 /tmp/passwd_back
salt-minion01:
    -rw-r--r-- 1 root root 2098 May 15 16:21 /tmp/passwd_back
```
`file.directory` 建立目录
```
[root@salt-master ~]# cat /srv/salt/base/directory.sls 
/tmp/saltdir:
  file.directory:
    - user: root
    - group: root
    - mode: 755
    - makedirs: True   #如果上一级目录不存在自动创建；类似（mkdir -p）

[root@salt-master ~]# salt '*' state.sls directory
[root@salt-master ~]# salt '*' cmd.run "ls -d /tmp/saltdir"
salt-minion03:
    /tmp/saltdir
salt-minion02:
    /tmp/saltdir
salt-minion01:
    /tmp/saltdir
```
`file.recurse` 下发整个目录
```
[root@salt-master ~]# cat /srv/salt/base/httpd_conf_dir.sls
httpd_conf_dir:
  file.recurse:
    - name: /etc/httpd/conf.d
    - source: salt://files/conf.d
    - file_mode: 600    #文件权限
    - dir_mode: 755    #目录权限
    - include_empty: True   #同步空目录
    - clean: True    #使用后minion与master保持一致

[root@salt-master ~]# rsync -avh /etc/httpd/conf.d /srv/salt/base/files/
```
`file.symlink` 建立软链接
```
[root@salt-master ~]# cat /srv/salt/base/target_link.sls 
/etc/grub.cfg:
  file.symlink:
    - target: /etc/grub2.cfg

[root@salt-master ~]# salt '*' state.sls target_link
[root@salt-master ~]# salt '*' cmd.run "ls -l /etc/grub.cfg"
salt-minion03:
    lrwxrwxrwx 1 root root 14 May 15 16:42 /etc/grub.cfg -> /etc/grub2.cfg
salt-minion01:
    lrwxrwxrwx 1 root root 14 May 15 16:42 /etc/grub.cfg -> /etc/grub2.cfg
salt-minion02:
    lrwxrwxrwx 1 root root 14 May 15 16:42 /etc/grub.cfg -> /etc/grub2.cfg
```
#### service服务模块
[service模块官档](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.service.html)
```
[root@salt-master ~]# cat /srv/salt/base/service_httpd.sls 
httpd:
  service.running:
    - name: httpd   #服务名称
    - enable: True    #开机自启动
    - reload: True    #允许重载配置文件，不写则是restart

或者这样写
[root@salt-master ~]# cat /srv/salt/base/service_httpd.sls 
httpd:   #即表示ID，又表示服务名
  service.running:
    - enable: True
    - reload: True
```
## 高级状态模块
>当我们想要不同的主机应用不同的配置，那么可以使用高级状态管理 `top file`
来进行管理。

## LAMP架构案例
说明：该案例在`prod`环境配置
1）环境准备，定义`file_roots`环境
```
[root@salt-master ~]# vim /etc/salt/master
file_roots:
  base:
    - /srv/salt/base
  dev:
    - /srv/salt/dev
  prod:
    - /srv/salt/prod
```
2）创建对应环境目录
```
[root@salt-master ~]# mkdir -p /srv/salt/{base,dev,prod}
[root@salt-master ~]# mkdir /srv/salt/prod/{httpd,php,mysql}
```
3）编写`state sls`状态文件
