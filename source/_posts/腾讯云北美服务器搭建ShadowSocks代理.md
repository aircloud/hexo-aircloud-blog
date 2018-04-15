---
title: 腾讯云北美服务器搭建ShadowSocks代理
date: 2016-08-08 19:15:01
tags:
    - ShadowSocks
---

注：本教程适合centos系列和red hat系列

登陆SSH 
新的VPS可以先升级

```
yum -y update
```

有些VPS 没有wget 
这种要先装

```
yum -y install wget
```

输入以下命令：（可以复制）

```
wget --no-check-certificate https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks.sh
chmod +x shadowsocks.sh
./shadowsocks.sh 2>&1 | tee shadowsocks.log
```

第一行是下载命令，下载东西，第二行是修改权限，第三行是安装命令

下面是按照配置图

```
配置：
密码：（默认是teddysun.com）
端口：默认是8989
然后按任意键安装，退出按 Ctrl+c
```

安装完成会有一个配置

```
Congratulations, shadowsocks install completed!Your Server IP:  ***** VPS的IP地址Your Server Port:  *****  你刚才设置的端口Your Password:  ****  你刚才设置的密码Your Local IP:  127.0.0.1 Your Local Port:  1080 Your Encryption Method:  aes-256-cfb Welcome to visit:https://teddysun.com/342.htmlEnjoy it!
```

然后即可以使用