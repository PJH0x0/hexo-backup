---
title: VPS搭建个人博客
tags:
- hexo
- 个人博客
- vps
categories: hexo
---
> 前言: 废了老大的劲,终于是将个人博客搭好了.分享一下经验以供大家参考.阅读本文需要的条件: 有使用过Mac或者Linux的经验,会vi的基本操作

# 环境

1. vps账号
2. vps操作系统:centos6,**很重要: centos6和centos7的命令不一样**
3. 本地系统:Mac OS. Ubuntu等类Unix系统也是类似的

# ssh连接vps
## 生成ssh秘钥
先到~/.ssh/目录下有没有*id_rsa,id_rsa.pub*两个文件,
如果没有的话输入以下命令,然后一路回车即可
``` Shell
ssh-keygen -t rsa
```

## 通过ssh连接vps
1. 使用以下命令拷贝id到本地, 默认端口号:22
``` Shell 
ssh-copy-id -p 端口号 root@ip_address
```
2. 输入VPS的root密码,**注意:不是VPS的密码**
3. 连接vps
``` Shell
ssh -p 端口号 root@ip_address
```
# 安装Hexo
可以参考[Hexo的文档](https://hexo.io/zh-cn/docs/),这里有两种方式:本地安装,VPS安装.
推荐通过本地安装,理由有三个:
1. vps磁盘空间和内存都很小,就不要去消耗了
2. 每次发布都要手动发布, 很麻烦
3. 用ssh连接vps真的很卡

# 配置vps
## 安装nginx
nginx是个静态的服务器应用,不安装nginx vps服务器无法被其他人访问,首先要通过ssh连接vps.我的vps系统是centos6的,无法直接安装nginx,需要对源进行处理,其他系统请自行上网搜索安装
```Shell
# 对源进行处理
rpm -ivh http://nginx.org/packages/centos/6/noarch/RPMS/nginx-release-centos-6-0.el6.ngx.noarch.rpm
# 安装nginx
yum install nginx -y
# 启动nginx服务(centos7 使用systemctl start nginx.service启动)
service nginx start
```
然后在本地系统打开浏览器, 然后输入vps服务器的ip地址,如果出现了**Welcome to
nginx**的字样则说明安装成功了

## 创建用于放置网页的目录
创建一个用来放置网页的目录,例如: 
```Shell
mkdir nginx;cd nginx;mkdir blog;cd blog
```
就可以创建一个nginx/blog目录,然后使用`pwd`命令获取当前的目录

## 编辑配置文件
执行`vi /etc/nginx/conf.d/default.conf`, 然后编辑
```Shell
server {
  root 网页目录; # 使用pwd命令获取的目录
  index index.html index.htm;#首页展示
  ...#一些不需要关注的配置
}
```
还有一个很重要的权限问题,因为我们登录的时候使用的是root权限,而nginx没有root权限,所以如果直接重启nginx刷新网页会报出403错误.
最简单的解决方案就是修改nginx的权限:`vi /etc/nginx/nginx.conf`然后将其中的user从nginx改为root, 
![hexo-root](nginx-config.png)
当然还有一种更好的方式,修改目录的权限
```
#这个命令我没有试过,使用时请自行斟酌
#第一个nginx是用户, 第二个nginx是我们创建的目录
chown -R -v nginx:root nginx
```
最后,执行`service nginx restart`,这时候如果刷新服务器的网页的话,还是会显示*403错误*,因为我们网页目录只是一个空目录,没有index文件,不急,我们慢慢来

## 安装rsync
这个是用来同步本地的网页到服务器目录的,有些人说vps上不安装也可以,但是我的不行.(其实不安装这个rsync也可以,后面有方法告诉大家怎么做)
```Shell
yum -y install rsync
```

# 配置Hexo
按照官网的提示装完Hexo之后,运行`hexo s`然后在浏览器中输入*localhost:4000*应该就可以有一个对应的*Hello World*界面了

## 安装rsync
```Shell
# 进入Hexo目录,然后执行
npm install hexo-deployer-rsync --save
```
## 配置_config.yml
主要配置以下几项:
```Shell
# 记得冒号后面一定要加空格
title: 
subtitle:
description: 
keywords:
author: 
language: zh
timezone: Asia/Shanghai
theme: next

# 下面的配置比较重要
deploy:
  type: rsync
  host: #ip_address
  user: root
  root: /root/nginx/blog/ # nginx的网页目录
  port: # 端口号
  delete: true
  verbose: true
  ignore_errors: false
```
这样就差不多了,接下来要做执行`hexo g;hexo d`这样就把网页部署到vps服务器上了,输入vps服务器的地址,应该就可以出来*Hello world*页面了

## 不使用rsync部署
其实rsync就是把Hexo下面的public目录的文件已到vps的网页下面的目录中去了而已.那我们手动移动也是可以的
```Shell
# 进入hexo目录, 这里使用我创建的目录作为例子
scp -r public root@ip_address:/root/nginx/blog/ 
```


