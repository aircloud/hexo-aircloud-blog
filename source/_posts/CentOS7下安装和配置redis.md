---
title: CentOS7下安装和配置redis
date: 2016-10-4 23:52:11
tags:
    - centOS
    - redis
---
Redis是一个高性能的，开源key-value型数据库。是构建高性能，可扩展的Web应用的完美解决方案，可以内存存储亦可持久化存储。因为要使用跨进程，跨服务级别的数据缓存，在对比多个方案后，决定使用Redis。顺便整理下Redis的安装过程，以便查阅。


 1 . 下载Redis
目前，最新的Redist版本为3.0，使用wget下载，命令如下：
```

# wget http://download.redis.io/releases/redis-3.0.4.tar.gz

```
 2 . 解压Redis
下载完成后，使用tar命令解压下载文件：
```

# tar -xzvf redis-3.0.4.tar.gz
```
3 . 编译安装Redis
切换至程序目录，并执行make命令编译：
```
# cd redis-3.0.4
# make
```
执行安装命令
```
# make install
```
make install安装完成后，会在/usr/local/bin目录下生成下面几个可执行文件，它们的作用分别是：

* redis-server：Redis服务器端启动程序
* redis-cli：Redis客户端操作工具。也可以用telnet根据其纯文本协议来操作
* redis-benchmark：Redis性能测试工具
* redis-check-aof：数据修复工具
* redis-check-dump：检查导出工具

备注

有的机器会出现类似以下错误：
```
make[1]: Entering directory `/root/redis/src'
You need tcl 8.5 or newer in order to run the Redis test
……
```
这是因为没有安装tcl导致，yum安装即可：
```
yum install tcl
```
4 . 配置Redis
复制配置文件到/etc/目录：
```
# cp redis.conf /etc/
```
为了让Redis后台运行，一般还需要修改redis.conf文件：
```
vi /etc/redis.conf
```
修改daemonize配置项为yes，使Redis进程在后台运行：
```
daemonize yes
```
5 . 启动Redis
配置完成后，启动Redis：
```
# cd /usr/local/bin
# ./redis-server /etc/redis.conf
```
检查启动情况：
```
# ps -ef | grep redis
```
看到类似下面的一行，表示启动成功：
```
root     18443     1  0 13:05 ?        00:00:00 ./redis-server *:6379 
```
6 . 添加开机启动项
让Redis开机运行可以将其添加到rc.local文件，也可将添加为系统服务service。本文使用rc.local的方式，添加service请参考：Redis 配置为 Service 系统服务 。

为了能让Redis在服务器重启后自动启动，需要将启动命令写入开机启动项：
```
echo "/usr/local/bin/redis-server /etc/redis.conf" >>/etc/rc.local
```
7 . Redis配置参数
在 前面的操作中，我们用到了使Redis进程在后台运行的参数，下面介绍其它一些常用的Redis启动参数：
```
daemonize：是否以后台daemon方式运行
pidfile：pid文件位置
port：监听的端口号
timeout：请求超时时间
loglevel：log信息级别
logfile：log文件位置
databases：开启数据库的数量
save * *：保存快照的频率，第一个*表示多长时间，第三个*表示执行多少次写操作。在一定时间内执行一定数量的写操作时，自动保存快照。可设置多个条件。
rdbcompression：是否使用压缩
dbfilename：数据快照文件名（只是文件名）
dir：数据快照的保存目录（仅目录）
appendonly：是否开启appendonlylog，开启的话每次写操作会记一条log，这会提高数据抗风险能力，但影响效率。
appendfsync：appendonlylog如何同步到磁盘。三个选项，分别是每次写都强制调用fsync、每秒启用一次fsync、不调用fsync等待系统自己同步
```
