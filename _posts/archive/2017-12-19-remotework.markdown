---
layout: archive
title: 本地编码，远程调试运行
date: 2017-12-19 14:00
categories: tool
tag: vim rsync ssh

---
后台程序员一般都远程服务器上运行调试代码，也就有两个选择，ssh登录远程环境，直接在远程环境敲代码，调试编译运行方便，但是得重新配置远程环境，而且一般的远程环境软件版本比较old,  所以一些新的敲代码的辅助不能够很好的兼容了，所以有没有一种能在本地编写代码，远程编译运行调试的方法呢？在网上查找了一番资料后找到了这么个方法，主要利用ssh, rsync,来进行。

## 环境准备
debian/ubuntu 安装ssh, rsync
```
sudo apt-get install rsync ssh inotify-tools -y
```
redhat/centos系
```
sudo yum install rsync ssh inotify-tools -y
```

## ssh 配置

### 本地和远程开发机能够直连
如果本地和远程开发机能够直连，这当然能省不少事，配置起来也方便，
首先我们配置ssh免密码登录
> shell>> ssh-keygen

一路回车选定默认后在，~/.ssh/文件夹下有id_rsa私钥，id_rsa.pub公钥,只需将id_rsa.pub公钥放到远程开发机上即可，
> shell>> ssh-copy-id user@remote-host-name

这样之后就能够免密码输入了，为了再方便能够简化输入，我们在~/.ssh/config文件下添加
```
Host dev-name
HostName remote-host-name
Port 22
User user
```
这样直接键入 *ssh dev-name* 便可直接登录远程开发机
### 本地和远程开发机经过跳板机
如果远程开发机和本地连接要经过跳板机，这要如果要一键登录有两种方法,（默认ssh jump-host一键登录跳板机，在跳板机上ssh dev-name一键登录开发机）
1. ssh -tt jump-host ssh -tt dev-name

这个命令是先使用本地~/.ssh/config的配置登录跳板机，再使用跳板机上的~/.ssh/config配置登录开发机，所以我这样的话我们只需要将本地的ssh公钥放到跳板机上，跳板机的公钥放到开发机上。

2. 配置~/.ssh/config

```
Host jump-host
HostName jump-host-name
Port 22
User user

Host dev-name
HostName remote-host-name
Port 22
User user
ProxyCommand ssh jump-host nc %h %p
```
上部分配置本地到跳板机能都一键直连，下半部分中ProxyCommand命令把跳板机设置成代理，本地连接开发机要先链接跳板机，再连接开发机，过程中都使用本地的私钥去验证登录，所以我们必须将本地的公钥放在跳板机和开发机上(TODO: 能不能向第一种方法那样只将本地的公钥只需放在跳板机上而不必再放在开发机上，感觉应该有方法，但目前没有找到，后续在看看)。
这样配置后我们就可以直接键入 ssh dev-name登录开发机了

## rsync使用
rsync的主要功能使用来远程同步的，具体使用暂且不细聊，我们rsync能够同步本地的代码到远程，但是能我们肯定不能每修改一次代码就敲一次命令同步，这要的话就重复的工作太多，我们需要的效果是我一修改代码，它就能够自动同步上去。要实现这要的效果，有一个命令能帮助我们 inotifywait, 该命令能够监听文件的变动的一系列动作，我们可以使用一个脚本，让它后台运行，监听动作并同步
```
#!/bin/bash
# 监听文件并同步代码

# REMOTE的格式如"dev-name:/project/dir/", dev-name是配置在ssh里面的主机名
# LOCAL的格式是文件夹名
REMOTE=$1
LOCAL=$2

# rsync from remote
rsync -avz $REMOTE ./

# loop to rsync to remote
while true;do
    inotifywait inotifywait -r -e modify,attrib,close_write,move,create,delete $LOCAL
    rsync -avz $LOCAL $REMOTE
done

```
如此一来大功告成，我们就能够开心的在本地敲代码了
