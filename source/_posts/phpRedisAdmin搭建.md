---
title: phpRedisAdmin搭建
date: 2019-04-26 16:30:09
tags: Linux运维日常
categories: Linux运维日常
copyright: true
---
>在已有的` lnmp` 环境下搭建 `phpRedisAdmin` 由于服务器做了限制，隧道端口转发等都不能使用，导致 `RedisDesktopManager` 管理工具不能使用，只能通过 `phpRedisAdmin`来进行管理了。

1）下载软件
```
# git clone https://github.com/ErikDubbelboer/phpRedisAdmin.git
# cd phpRedisAdmin
# git clone https://github.com/nrk/predis.git vendor
```
2）放置到`web`站点目录
```
# cd ..
# mv phpRedisAdmin /opt/
# chown apache.apache phpRedisAdmin/ -R
```
3）编辑`nginx`配置文件
```
# vim phpRedisAdmin.conf
server {
    listen       80;
    server_name  www.redis.com;
    autoindex off;

    error_log  logs/redis_error.log error;
    access_log logs/redis_access.log main;

    location / {
    root /opt/phpRedisAdmin;
        index  index.php index.html index.htm;
    }

    fastcgi_intercept_errors on;
    error_page  404             /404.html;
    location ~ \.php$ {
        root           html;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /opt/phpRedisAdmin$fastcgi_script_name;
        include        fastcgi_params;
    }

    location ~ /\. {
        deny all;
        error_page  403             /404.html;
    }
}
```
4）配置连接管理，默认连接的是`6379`端口
```
# vim /opt/phpRedisAdmin/includes/config.sample.inc.php
$config = array(
  'servers' => array(
    array(
      'name'   => 'local server', // Optional name.
      'host'   => '127.0.0.1',
      'port'   => 6395,
      'filter' => '*',
      'scheme' => 'tcp', // Optional. Connection scheme. 'tcp' - for TCP connection, 'unix' - for connection by unix domain socket
      'path'   => '', // Optional. Path to unix domain socket. Uses only if 'scheme' => 'unix'. Example: '/var/run/redis/redis.sock'
      'auth' => 'jieg6uy5Dach0yooxo1l' // Warning: The password is sent in plain-text to the Redis server.
    ),
```
5）`windows`添加`host`然后访问 `http://www.redis.com`
![](https://upload-images.jianshu.io/upload_images/11763553-53b72098e4fac4ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


