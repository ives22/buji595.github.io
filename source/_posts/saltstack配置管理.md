---
title: saltstack配置管理
date: 2019-05-15 23:42:05
tags: Saltstack
categories: Saltstack
copyright: true
---
## Saltstack状态模块
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
### pkg软件模块
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
### file文件模块
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
### service服务模块
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
来进行管理。可以通过正则，`grain`模块，或分组名，来进行匹配，再下一级是要执行的`state`文件

可以将我们的配置需求转换为`YAML`并在`Top file`文件中表示：
![](saltstack配置管理/01.png)

`Top file`示例
```
base:
  '*': #通过正则去匹配所有minion
    - app.nginx

  webserver: #定义的分组名称
    - match: nodegroup
    - app.cron

  'os:centos':  #通过grains模块匹配
    - match: grains
    - nginx 
```
`Top file` 高级状态的执行
```
[root@salt-master ~]# salt '*' state.highstate
```
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
[root@salt-master ~]# mkdir /srv/salt/prod/{httpd,php,mysql,files}
```
3）配置文件准备及测试文件准备
```
[root@salt-master ~]# cp /etc/my.cnf /srv/salt/prod/files/
[root@salt-master ~]# cp /etc/httpd/conf/httpd.conf /srv/salt/prod/files/
[root@salt-master ~]# cp /etc/php.ini  /srv/salt/prod/files/
[root@salt-master ~]# echo "<h1>LAMP html</h1>" >>/srv/salt/prod/files/index.html
[root@salt-master ~]# echo "<?php phpinfo(); ?>" >> /srv/salt/prod/files/index.php
```
4）编写`state sls`状态文件
```
#httpd
[root@salt-master ~]# cat /srv/salt/prod/httpd/init.sls
apache-install:
  pkg.installed:
    - pkgs:
      - httpd
      - httpd-tools

apache-config:
  file.managed:
    - name: /etc/httpd/conf/httpd.conf
    - source: salt://files/httpd.conf
    - user: root
    - group: root
    - mode: 644

apache-service:
  service.running:
    - name: httpd
    - enable: True

#php
[root@salt-master ~]# cat /srv/salt/prod/php/init.sls
php-install:
  pkg.installed:
    - pkgs:
      - php
      - php-mysql
      - php-pdo
      - php-cli

php-config:
  file.managed:
    - name: /etc/php.ini
    - source: salt://files/php.ini
    - user: root
    - group: root
    - mode: 644

#mysql
[root@salt-master ~]# cat /srv/salt/prod/mysql/init.sls
mariadb-install:
  pkg.installed:
    - pkgs:
      - mariadb-server
      - mariadb

mariadb-config:
  file.managed:
    - name: /etc/my.cnf
    - source: salt://files/my.cnf
    - user: root
    - group: root
    - mode: 644

mariadb-service:
  service.running:
    - name: mariadb
    - enable: True

#测试文件
[root@salt-master ~]# cat /srv/salt/prod/testfile.sls
/var/www/html/index.html:
  file.managed:
    - source: salt://files/index.html

/var/www/html/index.php:
  file.managed:
    - source: salt://files/index.php
```
6）`topfile`文件编写
```
[root@salt-master ~]# cat /srv/salt/base/top.sls
prod:
  'salt-minion*':
    - httpd.init
    - php.init
    - mysql.init
    - testfile
```
7）部署`LAMP`整体`state`文件查看
```
[root@salt-master ~]# tree /srv/salt/
/srv/salt/
├── base
│   └── top.sls
├── dev
└── prod
    ├── files
    │   ├── httpd.conf
    │   ├── index.html
    │   ├── index.php
    │   ├── my.cnf
    │   └── php.ini
    ├── httpd
    │   └── init.sls
    ├── mysql
    │   └── init.sls
    ├── php
    │   └── init.sls
    └── testfile.sls
```
8）执行`topfile`
```
[root@salt-master ~]# salt '*' state.highstate
```
## States状态依赖
通过上面的`lamp`可以看出已经可以使用`state`模块来定义`minion`的状态了，但是如果一个主机涉及多个状态，并且状态之间相互关联，在执行顺序上有先后之分，那么必须引用`requisites`来进行控制
>关系说明：
1、`require` 我依赖某个状态，我依赖谁
2、`require_in` 我被某个状态依赖，谁依赖我
3、`watch` 我关注某个状态，当状态发生改变，进行`restart`或者`reload`操作
4、`watch_in` 我被某个状态关注
5、`include` 我引用谁

1）修改上面`lamp`状态间依赖关系
```
#httpd
[root@salt-master ~]# cat /srv/salt/prod/httpd/init.sls
apache-install:
  pkg.installed:
    - pkgs:
      - httpd
      - httpd-tools

apache-config:
  file.managed:
    - name: /etc/httpd/conf/httpd.conf
    - source: salt://files/httpd.conf
    - user: root
    - group: root
    - mode: 644
    - require:
      - pkg: apache-install   #表示上面apache-install执行成功，才能执行apache-config

apache-service:
  service.running:
    - name: httpd
    - enable: True
    - require:
      - file: apache-config
    - watch:
      - file: apache-config

#php
[root@salt-master ~]# cat /srv/salt/prod/php/init.sls
php-install:
  pkg.installed:
    - pkgs:
      - php
      - php-mysql
      - php-pdo
      - php-cli
    - reqiure_in:
      - file: php-config

php-config:
  file.managed:
    - name: /etc/php.ini
    - source: salt://files/php.ini
    - user: root
    - group: root
    - mode: 644

#mysql
[root@salt-master ~]# cat /srv/salt/prod/mysql/init.sls
mariadb-install:
  pkg.installed:
    - pkgs:
      - mariadb-server
      - mariadb

mariadb-config:
  file.managed:
    - name: /etc/my.cnf
    - source: salt://files/my.cnf
    - user: root
    - group: root
    - mode: 644
    - require:
      - pkg: mariadb-install

mariadb-service:
  service.running:
    - name: mariadb
    - enable: True
    - reload: True
    - require:
      - file: mariadb-config
    - watch:
      - file: mariadb-config
```
2）修改引用关系后`include`
```
[root@salt-master ~]# tree /srv/salt/
/srv/salt/
├── base
│   └── top.sls
├── dev
└── prod
    ├── files
    │   ├── httpd.conf
    │   ├── index.html
    │   ├── index.php
    │   ├── my.cnf
    │   └── php.ini
    ├── httpd
    │   └── init.sls
    ├── lamp.sls
    ├── mysql
    │   └── init.sls
    ├── php
    │   └── init.sls
    └── testfile.sls

[root@salt-master ~]# cat /srv/salt/prod/lamp.sls
include:
  - httpd.init
  - php.init
  - mysql.init
  - testfile

[root@salt-master ~]# cat /srv/salt/base/top.sls
prod:
  'salt-minion*':
    - lamp
```
3）编写`SLS`技巧
>1、按照状态分类，如果单独使用，清晰明了
2、按照服务分类，可以被其它`SLS`引用

## Jinja模板使用
>配置文件一般灵活多变，比如配置`apache`的`IP`地址或者端口`PORT`等，则可以动态传值。
[Jinja官档](http://docs.jinkan.org/docs/jinja2/)
[salt jinja官档](https://docs.saltstack.com/en/latest/topics/jinja/index.html)

`Jinja2` 模板包含变量和表达式，变量用 &#123;&#123; ... &#125;&#125; 包围，表达式用 &#123;&#37; ... &#37;&#125; 包围。变量使用示例：
```
[root@salt-master ~]# cat /srv/salt/base/var.sls 
{% set var= 'hello world!' %}
test_var:
  cmd.run:
    - name: echo "测试变量 {{ var }}"

[root@salt-master ~]# salt 'salt-minion01' state.sls var
salt-minion01:
----------
          ID: test_var
    Function: cmd.run
        Name: echo "测试变量 hello world!"
      Result: True
     Comment: Command "echo "测试变量 hello world!"" run
     Started: 14:50:58.302424
    Duration: 12.358 ms
     Changes:   
              ----------
              pid:
                  22510
              retcode:
                  0
              stderr:
              stdout:
                  测试变量 hello world!

Summary for salt-minion01
------------
Succeeded: 1 (changed=1)
Failed:    0
------------
Total states run:     1
Total run time:  12.358 ms
```
`jinja2` 常用变量
1、字符串类型
```
{% set var = 'test' %}  #定义变量
{{ var }}  #调用变量
```
2、列表类型
```
{% set list = ['one', 'two', 'three'] %}
{{ list[1] }}  #获取变量的第一个值
```
3、字典类型
```
{% set dict = {'key1':'value1', 'key2':'value2'} %}
{{ dict['key1'] }}  #获取'key1'的值
```
示例1：`Saltstack`使用`jinja`模块配置`apache`监听端口
```
#1.告诉file状态模块，需要使用jinja
    - template: jinja
#2.列出参数列表
    - defaults:
      PORT: 8000
#3.配置文件引用jinja模板
{{ PORT }}

# 配置示例
apache-config:
  file.managed:
    - name: /etc/httpd/conf/httpd.conf
    - source: salt://files/httpd.conf
    - user: root
    - group: root
    - mode: 644
    - template: jinja
    - defaults:
      PORT: 8000

# 修改httpd.conf配置文件引用变量
Listen {{ PORT }}
```
示例2：使用`grinas` 方式进行赋值
```
#配置示例
apache-config:
  file.managed:
    - name: /etc/httpd/conf/httpd.conf
    - source: salt://files/httpd.conf
    - user: root
    - group: root
    - mode: 644
    - template: jinja
    - defaults:
      PORT: 8000
      IPADDR: {{ grains['fqdn_ip4'][0] }}

# 修改httpd.conf配置文件引用变量
Listen {{ IPADDR }}:{{ PORT }}
```
示例3：通过`jinja+grains`根据系统不同安装`apache`
```
[root@salt-master ~]# cat /srv/salt/base/httpd.sls
#根据grains获取的值判别系统后安装软件
httpd-install:
  pkg.installed:
{% if grains['os'] == 'CentOS' %}
    - name: httpd
{% elif grains['OS'] == 'Debin' %}
    - name: apache2
{% endif %}
```


