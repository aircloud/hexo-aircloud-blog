---
title: Linux服务器初始化设置用户和ssh公私钥登陆
date: 2018-04-11 17:16:37
tags:
    - Linux
    - ssh
---

>当我们开始使用一个新的服务器的时候，首先一定要对服务器的登陆等做一些修改工作，笔者曾经就因为对服务器登陆安全没有重视，导致服务器数据全部丢失。接下来我们按照步骤，罗列出应该做的一些事情。

### 修改ssh端口号

第一件事情：

修改ssh端口号： 之后加上一个端口比如说50000

`vi /etc/ssh/sshd_config`之后在port字段加上一个端口比如说50000，原来的端口号字段可能是被注释掉的，要先解除注释。

然后执行：

```
service sshd restart
```

这个时候可能还要重新配置一下防火墙，开放50000端口，具体如何配置也可以参考[这里](https://blog.csdn.net/ul646691993/article/details/52104082)的后半部分。但是目前，阿里云的服务器实测是不需要再配置防火墙的，但是需要去登陆到网页后台修改安全组。

之后就可以通过这样的方式登录了：(注意登录方式一定要写对)

```shell
ssh root@115.29.102.81 -p 50000
```

### 创建用户

这个时候我们还是用root进行操作，所以我们接下来要给自己创建一个账户，比如创建一个如下的用户：

```
useradd xiaotao
passwd xiaotao
```

可以用`ls -al /home/``查看一下账户

对创建的这个用户增加sudo权限： 相关配置文件/etc/sudoers中，但是这个文件是只读的，所以要更改一下权限

```
chmod u+w sudoers
```

然后进入这个文件在这里进行更改：

```
root    ALL=(ALL)       ALL
xiaotao  ALL=(ALL)       ALL
```

然后再改回权限：

```
chmod u-w sudoers
```

注意一点，CentOS 7预设容许任何帐号透过ssh登入（也就是说自己根本不用改改，直接新建帐号登录即可），包括根和一般帐号，为了不受根帐号被黑客暴力入侵，我们必须禁止 root帐号的ssh功能，事实上root也没有必要ssh登入伺服器，因为只要使用su或sudo（当然需要输入root的密码）普通帐号便可以拥有root的权限。使用vim（或任何文本编辑器）开启的/ etc/ SSH/ sshd_config中，寻找：

```
＃PermitRootLogin yes
```
修改：

```
PermitRootLogin no
```

### 配置公私钥加密登录

**这一步骤要切换到自己新建的用户，不能再用 root 用户了，否则可能无法正常登陆。**

很多时候以上所说的还是不够安全，为了更加安全方便，我们采用公私钥对称加密登录，简单的讲做法就是再客户端生成一把私钥一把公钥，私钥是在客户端的，公钥上传到服务端，对称加密进行登录。

在客户端先进到这个目录：

```
cd ~/.ssh
```

生成公钥和私钥（实际上如果之前有的话就不用重新生成了）

```
ssh-keygen -t rsa
```

接下来把公钥上传到服务端

```
scp ~/.ssh/id_rsa.pub xiaotao@<ssh_server_ip>:~
```

在服务端执行以下命令(如果没有相关的文件和文件夹要先进行创建，注意不要使用 sudo )

```
cat  id_rsa.pub >> ～/.ssh/authorized_keys
```

配置服务器的/etc/ssh/sshd_config，下面是一些建议的配置：

```
vim /etc/ssh/sshd_config
# 禁用root账户登录，非必要，但为了安全性，请配置
PermitRootLogin no

# 是否让 sshd 去检查用户家目录或相关档案的权限数据，
# 这是为了担心使用者将某些重要档案的权限设错，可能会导致一些问题所致。
# 例如使用者的 ~.ssh/ 权限设错时，某些特殊情况下会不许用户登入
StrictModes no

# 是否允许用户自行使用成对的密钥系统进行登入行为，仅针对 version 2。
# 至于自制的公钥数据就放置于用户家目录下的 .ssh/authorized_keys 内
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile      %h/.ssh/authorized_keys

#有了证书登录了，就禁用密码登录吧，安全要紧
PasswordAuthentication no
```

然后不要忘记 `sudo service sshd restart`


一般来讲，这样就算是成功了，我们可以在客户端尝试：

```
ssh -i ~/.ssh/id_rsa remote_username@remote_ip
```

如果不行，可能是服务端或客户端相关 `.ssh` 文件权限不对，可以进行如下尝试：

```
服务端
chown -R 0700  ~/.ssh
chown -R 0644  ~/.ssh/authorized_keys

客户端改一下
chmod 600 id_rsa
```