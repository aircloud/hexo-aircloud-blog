---
title: CentOS安装node8.x版本
date: 2017-12-15 12:52:34
tags:
    - centOS
    - node
---
### CentOS 安装 node 8.x 版本

由于一些原因需要给CentOS服务器安装8.0以上版本的node, 本来直接通过yum管理安装管理，但是没找到好办法，在此记录一下自己最后使用的简单过程：

安装之前删除原来的node和npm (我原来是用yum安装的，如果是第一次安装可以省略这一步):

```
yum remove nodejs npm -y
```

首先我们随便进入服务器的一个目录，然后从淘宝的源拉取内容:

```
wget https://npm.taobao.org/mirrors/node/v8.0.0/node-v8.0.0-linux-x64.tar.xz 
```

解压缩:

```
sudo tar -xvf node-v8.0.0-linux-x64.tar.xz 
```

进入解压目录下的 bin 目录，执行 ls 命令

```
cd node-v8.0.0-linux-x64/bin && ls 
```

我们发现有node 和 npm

这个时候我们测试:

```
./node -v
```

这个时候我们发现实际上已经安装好了，接下来就是要建立链接文件。

这里还是，如果我们之前已经安装过了，那么我们要先删除之前建立的链接文件：

```
sudo rm -rf /usr/bin/node
sudo rm -rf /usr/bin/npm
```

然后建立链接文件:

```
sudo ln -s /usr/share/node-v8.0.0-linux-x64/bin/node /usr/bin/node
sudo ln -s /usr/share/node-v8.0.0-linux-x64/bin/npm /usr/bin/npm
```

注意这里的第一个路径不要直接复制粘贴，要写当前文件的真正的路径，这个可以通过pwd获取。

然后我们可以通过`node -v`等测试已经安装成功。
