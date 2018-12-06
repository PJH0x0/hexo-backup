---
title: VPS搭建个人博客
tags: Hexo
categories: hexo
---
> 前言: 废了老大的劲,终于是将个人博客搭好了.分享一下经验以供大家参考

## 前提环境
1. VPS账号
2. VPS操作系统:centos6,**很重要: centos6和centos7的命令不一样**
3. 本地系统:Mac OS. Ubuntu等类Unix系统也是类似的

## SSH连接VPS
#### 1. 生成SSH秘钥
先到~/.ssh/目录下有没有*id_rsa,id_rsa.pub*两个文件,
如果没有的话输入以下命令,然后一路回车即可
```Shell
ssh-keygen -t rsa
```
#### 2. 通过SSH连接VPS
1. 输入```ssh-copy-id -p 端口号 root@ip_address```, 默认端口号:22
2. 输入VPS的root密码,**注意:不是VPS的密码**
3. 最后通过```ssh -p 端口号 root@ip_address``` 连接vps

## 安装Hexo
可以参考[Hexo的文档](https://hexo.io/zh-cn/docs/),这里有两种方式:本地安装,VPS安装.
推荐通过本地安装,理由有两个:
1. VPS磁盘空间和内存都很小,就不要去消耗了
2. 每次发布都要手动发布, 很麻烦
3. 

