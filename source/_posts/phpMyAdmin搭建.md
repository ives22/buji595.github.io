---
title: phpMyAdmin搭建
date: 2019-04-26 15:30:09
tags: Linux运维日常
categories: Linux运维日常
copyright: true
---
##### 在已有的 lnmp 环境下搭建 phpMyAdmin。 由于服务器做了限制，隧道端口转发等都不能使用，导致 Navicat 管理工具不能使用，只能通过 phpMyAdmin 来进行管理了。

- 下载phpMyAdmin\
官网地址：https://www.phpmyadmin.net/

- 上传到服务器进行安装
```html
# unzip phpMyAdmin-4.8.5-all-languages.zip
# mv phpMyAdmin-4.8.5-all-languages /opt/phpMyAdmin
# chown apache. /opt/phpMyAdmin
```
- 编辑配置文件 config.default.php
```
# vim /opt/phpMyAdmin/libraries/config.default.php
$cfg['Servers'][$i]['host'] = '130.39.113.45';
$cfg['Servers'][$i]['port'] = '3306';
$cfg['Servers'][$i]['socket'] = 'socket';
$cfg['Servers'][$i]['user'] = 'tj_court';
$cfg['Servers'][$i]['password'] = 'AigheiguSh4eesh0eey8';
```
- 配置nginx站点
```html
# vim phpMyAdmin.conf
server {
    listen       80;
    server_name  www.phpadmin.com;
    autoindex off;

    error_log  logs/phpadmin_error.log error;
    access_log logs/phpadmin_access.log main;

    location / {
    root /opt/phpMyAdmin;
        index  index.php index.html index.htm;
    }

    fastcgi_intercept_errors on;
    error_page  404             /404.html;
    location ~ \.php$ {
        root           html;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /opt/phpMyAdmin$fastcgi_script_name;
        include        fastcgi_params;
    }

    location ~ /\. {
        deny all;
        error_page  403             /404.html;
    }
}
```
- 配置完成后启动nginx访问配置的域名进行连接
![](phpMyAdmin搭建/4.png)
![](phpMyAdmin搭建/6.png)

