---
title: Ansible之Roles
date: 2019-06-03 10:35:22
tags: Ansible
categories: Ansible
copyright: true
---
## Roles介绍
> `ansible`自`1.2`版本引入的新特性，用于层次性、结构化地组织`playbook`。`roles`能够根据层次型结构自动装载变量文件、`tasks`以及`handlers`等。要使用`roles`只需要在`playbook`中使用`include`指令引入即可。简单来讲，`roles`就是通过分别将变量、文件、任务、模板及处理器放置于单独的目录中，并可以便捷的`include`它们的一种机制。角色一般用于基于主机构建服务的场景中，但也可以是用于构建守护进程等场景中。主要使用场景代码复用度较高的情况下。

## Roles目录结构
![rolesdir.png](Ansible之Roles/rolesdir.png)

**各目录含义解释**
```
roles:          <--所有的角色必须放在roles目录下，这个目录可以自定义位置，默认的位置在/etc/ansible/roles
  project:      <---具体的角色项目名称，比如nginx、tomcat、php
    files：     <--用来存放由copy模块或script模块调用的文件。
    templates：	<--用来存放jinjia2模板，template模块会自动在此目录中寻找jinjia2模板文件。
    tasks：     <--此目录应当包含一个main.yml文件，用于定义此角色的任务列表，此文件可以使用include包含其它的位于此目录的task文件。
	  main.yml
    handlers：	<--此目录应当包含一个main.yml文件，用于定义此角色中触发条件时执行的动作。
	  main.yml
    vars：	 <--此目录应当包含一个main.yml文件，用于定义此角色用到的变量。
	  main.yml
    defaults：	<--此目录应当包含一个main.yml文件，用于为当前角色设定默认变量。
	  main.yml
    meta：	<--此目录应当包含一个main.yml文件，用于定义此角色的特殊设定及其依赖关系。
	  main.yml
```
## Roles示例
>通过`ansible roles`安装配置`httpd`服务，此处的`roles`使用默认的路径`/etc/ansible/roles`

1）创建目录
```
[root@ansible ~]# cd /etc/ansible/roles/
# 创建需要用到的目录
[root@ansible roles]# mkdir -p httpd/{handlers,tasks,templates,vars}
[root@ansible roles]# cd httpd/
[root@ansible httpd]# tree .
.
├── handlers
├── tasks
├── templates
└── vars

4 directories, 0 file
```
2）变量文件准备`vars/main.yml`
```
[root@ansible httpd]# vim vars/main.yml
PORT: 8088        #指定httpd监听的端口
USERNAME: www     #指定httpd运行用户
GROUPNAME: www    #指定httpd运行组
```
3）配置文件模板准备`templates/httpd.conf.j2`
```
# copy一个本地的配置文件放在templates/下并已j2为后缀
[root@ansible httpd]# cp /etc/httpd/conf/httpd.conf templates/httpd.conf.j2

# 进行一些修改，调用上面定义的变量
[root@ansible httpd]# vim templates/httpd.conf.j2
Listen {{ PORT }} 
User {{ USERNAME }}
Group {{ GROUPNAME }}
```
4）任务剧本编写，创建用户、创建组、安装软件、配置、启动等
```
# 创建组的task
[root@ansible httpd]# vim tasks/group.yml
- name: Create a Startup Group
  group: name=www gid=60 system=yes

# 创建用户的task
[root@ansible httpd]# vim tasks/user.yml
- name: Create Startup Users
  user: name=www uid=60 system=yes shell=/sbin/nologin

# 安装软件的task
[root@ansible httpd]# vim tasks/install.yml
- name: Install Package Httpd
  yum: name=httpd state=installed

# 配置软件的task
[root@ansible httpd]# vim tasks/config.yml
- name: Copy Httpd Template File
  template: src=httpd.conf.j2 dest=/etc/httpd/conf/httpd.conf
  notify: Restart Httpd

# 启动软件的task
[root@ansible httpd]# vim tasks/start.yml
- name: Start Httpd Service
  service: name=httpd state=started enabled=yes

# 编写main.yml，将上面的这些task引入进来
[root@ansible httpd]# vim tasks/main.yml
- include: group.yml
- include: user.yml
- include: install.yml
- include: config.yml
- include: start.ym
```
5）编写重启httpd的`handlers`，`handlers/main.yml`
```
[root@ansible httpd]# vim handlers/main.yml
# 这里的名字需要和task中的notify保持一致
- name: Restart Httpd
  service: name=httpd state=restarted
```
6）编写主的`httpd_roles.yml`文件调用httpd角色
```
[root@ansible httpd]# cd ..
[root@ansible roles]# vim httpd_roles.yml
---
- hosts: all
  remote_user: root
  roles:
    - role: httpd        #指定角色名称
```
7）整体的一个目录结构查看
```
[root@ansible roles]# tree .
.
├── httpd
│   ├── handlers
│   │   └── main.yml
│   ├── tasks
│   │   ├── config.yml
│   │   ├── group.yml
│   │   ├── install.yml
│   │   ├── main.yml
│   │   ├── start.yml
│   │   └── user.yml
│   ├── templates
│   │   └── httpd.conf.j2
│   └── vars
│       └── main.yml
└── httpd_roles.yml

5 directories, 10 files
```
8）测试`playbook`语法是否正确
```
[root@ansible roles]# ansible-playbook -C httpd_roles.yml 

PLAY [all] **************************************************************************************************

TASK [Gathering Facts] **************************************************************************************
ok: [192.168.1.33]
ok: [192.168.1.32]
ok: [192.168.1.31]
ok: [192.168.1.36]

TASK [httpd : Create a Startup Group] ***********************************************************************
changed: [192.168.1.31]
changed: [192.168.1.33]
changed: [192.168.1.36]
changed: [192.168.1.32]

TASK [httpd : Create Startup Users] *************************************************************************
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.31]
changed: [192.168.1.36]

TASK [httpd : Install Package Httpd] ************************************************************************
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.31]
changed: [192.168.1.36]

TASK [httpd : Copy Httpd Template File] *********************************************************************
changed: [192.168.1.33]
changed: [192.168.1.36]
changed: [192.168.1.32]
changed: [192.168.1.31]

TASK [httpd : Start Httpd Service] **************************************************************************
changed: [192.168.1.36]
changed: [192.168.1.31]
changed: [192.168.1.32]
changed: [192.168.1.33]

RUNNING HANDLER [httpd : Restart Httpd] *********************************************************************
changed: [192.168.1.36]
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.31]

PLAY RECAP **************************************************************************************************
192.168.1.31               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.32               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.33               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.36               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```
9）上面的测试没有问题，正式执行`playbook`
```
[root@ansible roles]# ansible-playbook -C httpd_roles.yml 

PLAY [all] **************************************************************************************************

TASK [Gathering Facts] **************************************************************************************
ok: [192.168.1.33]
ok: [192.168.1.32]
ok: [192.168.1.31]
ok: [192.168.1.36]

TASK [httpd : Create a Startup Group] ***********************************************************************
changed: [192.168.1.31]
changed: [192.168.1.33]
changed: [192.168.1.36]
changed: [192.168.1.32]

TASK [httpd : Create Startup Users] *************************************************************************
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.31]
changed: [192.168.1.36]

TASK [httpd : Install Package Httpd] ************************************************************************
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.31]
changed: [192.168.1.36]

TASK [httpd : Copy Httpd Template File] *********************************************************************
changed: [192.168.1.33]
changed: [192.168.1.36]
changed: [192.168.1.32]
changed: [192.168.1.31]

TASK [httpd : Start Httpd Service] **************************************************************************
changed: [192.168.1.36]
changed: [192.168.1.31]
changed: [192.168.1.32]
changed: [192.168.1.33]

RUNNING HANDLER [httpd : Restart Httpd] *********************************************************************
changed: [192.168.1.36]
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.31]

PLAY RECAP **************************************************************************************************
192.168.1.31               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.32               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.33               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.36               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[root@ansible roles]# ansible-playbook httpd_roles.yml 

PLAY [all] **************************************************************************************************

TASK [Gathering Facts] **************************************************************************************
ok: [192.168.1.32]
ok: [192.168.1.33]
ok: [192.168.1.31]
ok: [192.168.1.36]

TASK [httpd : Create a Startup Group] ***********************************************************************
changed: [192.168.1.32]
changed: [192.168.1.31]
changed: [192.168.1.33]
changed: [192.168.1.36]

TASK [httpd : Create Startup Users] *************************************************************************
changed: [192.168.1.31]
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.36]

TASK [httpd : Install Package Httpd] ************************************************************************
changed: [192.168.1.31]
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.36]

TASK [httpd : Copy Httpd Template File] *********************************************************************
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.31]
changed: [192.168.1.36]

TASK [httpd : Start Httpd Service] **************************************************************************
fatal: [192.168.1.36]: FAILED! => {"changed": false, "msg": "httpd: Syntax error on line 56 of /etc/httpd/conf/httpd.conf: Include directory '/etc/httpd/conf.modules.d' not found\n"}
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.31]

RUNNING HANDLER [httpd : Restart Httpd] *********************************************************************
changed: [192.168.1.33]
changed: [192.168.1.32]
changed: [192.168.1.31]

PLAY RECAP **************************************************************************************************
192.168.1.31               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.32               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.33               : ok=7    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.1.36               : ok=5    changed=4    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```
这里有看到报错，其原因是因为`192.168.1.36`这台机器是`centos6`系统，别的都是`centos7`系统，两个系统安装的`httpd`默认版本是不一样的，所以报错。如果需要优化。[参考这里](https://buji595.github.io/2019/05/29/Ansible%E4%B9%8BPlaybook/#template%E4%B9%8Bwhen)
## ansible roles总结
>1、编写任务(`task`)的时候，里面不需要写需要执行的主机，单纯的写某个任务是干什么的即可，装软件的就是装软件的，启动的就是启动的。单独做某一件事即可，最后通过`main.yml`将这些单独的任务安装执行顺序`include`进来即可，这样方便维护且一目了然。
>2、定义变量时候直接安装`k:v`格式将变量写在`vars/main.yml`文件即可，然后`task`或者`template`直接调用即可，会自动去`vars/main.yml`文件里面去找。
>3、定义`handlers`时候，直接在`handlers/main.yml`文件中写需要做什么事情即可，多可的话可以全部写在该文件里面，也可以像`task`那样分开来写，通过`include`引入一样的可以。在`task`调用`notify`时直接写与`handlers`名字对应即可(二者必须高度一直)。
>4、模板文件一样放在`templates`目录下即可，task调用的时后直接写文件名字即可，会自动去到`templates`里面找。注意：如果是一个角色调用另外一个角色的单个`task`时后，那么`task`中如果有些模板或者文件，就得写绝对路径了。