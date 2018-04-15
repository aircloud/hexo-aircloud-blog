---
layout: 阿里云服务器ecs
title: centOS7.2搭建nginx环境以及负载均衡
date: 2016-08-03 21:16:24
tags:
    - centOS
    - nginx
---
 之所以要整理出这篇文章，是因为1是搭建环境的过程中会遇到大大小小各种问题，2是网上目前也没有关于centos7.2搭建nginx环境的问题整理，因此在这里记录。

前置工作就不赘述了，首先`ssh root@115.29.102.81` (换成你们自己的公网IP)登陆进入到自己的服务器命令行，之后开始基本的安装：

**1.添加资源**

添加CentOS 7 Nginx yum资源库,打开终端,使用以下命令(没有换行):

```
sudo rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm

```

**2.安装Nginx**

在你的CentOS 7 服务器中使用yum命令从Nginx源服务器中获取来安装Nginx：
>*这里有一个需要注意的地方，尽量不要用网上的下载源码包然后再传到服务器上的方式进行安装，因为nginx已经不算是简单的Linux了，做了很多扩展，这个时候如果你用源码包安装会出现各种各样的问题，尽量用已经封装好的rpm\yum进行安装*
```
sudo yum install -y nginx
```
Nginx将完成安装在你的CentOS 7 服务器中。

**3.启动Nginx**

刚安装的Nginx不会自行启动。运行Nginx:
```
sudo systemctl start nginx.service
```
如果一切进展顺利的话，现在你可以通过你的域名或IP来访问你的Web页面来预览一下Nginx的默认页面

>当然，这里一般很可能会无法访问的。

我们先不急于解决我们的问题，先看看nginx的基本配置：


Nginx配置信息
```
网站文件存放默认目录

/usr/share/nginx/html
网站默认站点配置

/etc/nginx/conf.d/default.conf
自定义Nginx站点配置文件存放目录,自己在这里也可以定义别的名字的.conf，这个的作用以后再说。

/etc/nginx/conf.d/
Nginx全局配置

/etc/nginx/nginx.conf
在这里你可以改变设置用户运行Nginx守护程序进程一样,和工作进程的数量得到了Nginx正在运行,等等。
```
Linux查看公网IP

您可以运行以下命令来显示你的服务器的公共IP地址:(这个其实没用，不是公网IP)
```
ip addr show eth0 | grep inet | awk '{ print $2; }' | sed 's/\/.*$//'
```
___
好了，这个时候我们再来看看可能遇到的问题：无法在公网访问。

这个时候首先看看配置文件default.conf对不对，一个正确的例子：
(域名要先进行解析到响应的IP)
```
server {
    listen       80;
    server_name  nginx.310058.cn;

    #charset koi8-r;
    #access_log  /var/log/nginx/log/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

确定文件没问题了，看看这个时候是不是开启了nginx进程：

```
 ps -ef | grep nginx
```

应该会输出一个或者多个进程，如果没有的话就开启或者重启试试看。

这个时候接下来再试试在服务器上：
```
ping  115.29.102.81
telnet 115.29.102.81 80
wget nginx.310058.cn
```
如果有的命令没有就直接yum安装下:
```
yum -y install telnet
```
如果都可以的话，之后在本机尝试以上三行。如果没有命令也要安装下：
```
brew install wget
```

发现很可能本机telnet不通，而服务器telnet通。
这个时候就是**防火墙**的问题。

####centos7.2防火墙

由于centos 7版本以后默认使用firewalld后，网上关于iptables的设置方法已经不管用了，所以根本就别想用配置iptables做啥，根本没用。

查看下防火墙状态：
```
[root@iZ28dcsp7egZ conf.d]# systemctl status firewalld  
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2016-08-03 12:06:44 CST; 2h 49min ago
 Main PID: 424 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─424 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid

Aug 03 12:06:41 iZ28dcsp7egZ systemd[1]: Starting firewalld - dynamic firewall daemon...
Aug 03 12:06:44 iZ28dcsp7egZ systemd[1]: Started firewalld - dynamic firewall daemon.
```

增加80端口的权限：
```
firewall-cmd --zone=public --add-port=80/tcp --permanent  
```
 
 别忘了更新防火墙的配置：
```
firewall-cmd --reload
```
这个时候再`restart  nginx.service` 一下就会发现应该好了。


nginx 停止：

```
service nginx restart
也可以重启nginx

kill -QUIT 进程号  
#从容停止

kill -TERM 进程号
#或者
kill -INT 进程号
#快速停止

p-kill -9 nginx
强制停止

nginx -t 
#验证配置文件 前提是进入相应的配置的目录（自己实际测试的时候发现没有进入相应的配置目录也是可以的）

nginx -s reload
#重启

kill -HUP 进程号
#重启的另外一种方式
```

官方文档地址：
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Security_Guide/sec-Using_Firewalls.html#sec-Introduction_to_firewalld

附1:一个简单的负载均衡的实现:
weight默认是1，自己也可以更改。
```
upstream mypro {
				ip_hash;
                server 111.13.100.92 weight=2;
                server 183.232.41.1;
                server 42.156.140.7;
                }

        server {
                listen 8090;
                location / {
                proxy_pass http://mypro;
                }
        }

```


附2:防火墙基本学习：

``` 

1、firewalld简介
firewalld是centos7的一大特性，最大的好处有两个：支持动态更新，不用重启服务；第二个就是加入了防火墙的“zone”概念
 
firewalld有图形界面和工具界面，由于我在服务器上使用，图形界面请参照官方文档，本文以字符界面做介绍
 
firewalld的字符界面管理工具是 firewall-cmd 
 
firewalld默认配置文件有两个：/usr/lib/firewalld/ （系统配置，尽量不要修改）和 /etc/firewalld/ （用户配置地址）
 
zone概念：
硬件防火墙默认一般有三个区，firewalld引入这一概念系统默认存在以下区域（根据文档自己理解，如果有误请指正）：
drop：默认丢弃所有包
block：拒绝所有外部连接，允许内部发起的连接
public：指定外部连接可以进入
external：这个不太明白，功能上和上面相同，允许指定的外部连接
dmz：和硬件防火墙一样，受限制的公共连接可以进入
work：工作区，概念和workgoup一样，也是指定的外部连接允许
home：类似家庭组
internal：信任所有连接
对防火墙不算太熟悉，还没想明白public、external、dmz、work、home从功能上都需要自定义允许连接，具体使用上的区别还需高人指点
 
2、安装firewalld
root执行 # yum install firewalld firewall-config
 
3、运行、停止、禁用firewalld
启动：# systemctl start  firewalld
查看状态：# systemctl status firewalld 或者 firewall-cmd --state
停止：# systemctl disable firewalld
禁用：# systemctl stop firewalld
 
4、配置firewalld
查看版本：$ firewall-cmd --version
查看帮助：$ firewall-cmd --help
查看设置：
                显示状态：$ firewall-cmd --state
                查看区域信息: $ firewall-cmd --get-active-zones
                查看指定接口所属区域：$ firewall-cmd --get-zone-of-interface=eth0
拒绝所有包：# firewall-cmd --panic-on
取消拒绝状态：# firewall-cmd --panic-off
查看是否拒绝：$ firewall-cmd --query-panic
 
更新防火墙规则：# firewall-cmd --reload
                            # firewall-cmd --complete-reload
    两者的区别就是第一个无需断开连接，就是firewalld特性之一动态添加规则，第二个需要断开连接，类似重启服务
 
将接口添加到区域，默认接口都在public
# firewall-cmd --zone=public --add-interface=eth0
永久生效再加上 --permanent 然后reload防火墙
 
设置默认接口区域
# firewall-cmd --set-default-zone=public
立即生效无需重启
 
打开端口（貌似这个才最常用）
查看所有打开的端口：
# firewall-cmd --zone=dmz --list-ports
加入一个端口到区域：
# firewall-cmd --zone=dmz --add-port=8080/tcp
若要永久生效方法同上
 
打开一个服务，类似于将端口可视化，服务需要在配置文件中添加，/etc/firewalld 目录下有services文件夹，这个不详细说了，详情参考文档
# firewall-cmd --zone=work --add-service=smtp
 
移除服务
# firewall-cmd --zone=work --remove-service=smtp
 
还有端口转发功能、自定义复杂规则功能、lockdown，由于还没用到，以后再学习

```
