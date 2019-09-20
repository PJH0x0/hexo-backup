---
title: Ubuntu 18.04 LTS安装过程记录
date: 2019/09/20
categories:
  - tools
  - ubuntu
tags:
  - ubuntu
abbrlink: 2b951266
---

*硬件: 3700x + 2070super*
*系统: Winodows10 + Ubuntu18.04 双系统*

# 安装
安装完重新要进入BIOS修改启动方式, 用grub2.0进入, 不然进不去

# 修改源
这个首要任务, 最好是打开Software&Updates进行修改, 进入Software&Updates->Ubuntu software->Download from->other->China->选择一个国内的源. 因为网上的一些源可能不是18.04的, 要看清楚中间的bionic标识, 这里可以参考[Ubuntu 18.04换国内源](https://blog.csdn.net/xiangxianghehe/article/details/80112149)

# 安装N卡驱动
如果没有N卡驱动界面会非常卡, 几乎是显示不了, 而且调整不了分辨率.安装N卡驱动是参考的[Ubuntu 18.04 安装 NVIDIA 显卡驱动](https://zhuanlan.zhihu.com/p/59618999), 里面非常详细, 不过需要将2和1执行的顺序要调换一下
```shell
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
ubuntu-drivers devices
sudo ubuntu-drivers autoinstall
```
# 主题美化
到[Gnome-look](https://www.pling.com/s/Gnome)中找个最流行的就行, 在右下角有个Popularity的排序, 不知道为什么那么多人选择mac主题, 我觉得一般呐
[grub美化](https://tianyijian.github.io/2018/04/05/ubuntu-grub-beautify/)
[开机界面美化](https://maxusun.github.io/2018/04/05/Ubuntu%E4%BF%AE%E6%94%B9%E5%BC%95%E5%AF%BC%E7%95%8C%E9%9D%A2%E5%92%8C%E5%90%AF%E5%8A%A8%E7%95%8C%E9%9D%A2/index.html)

# grub设置
进入BIOS设置grub为第一启动项, 设置grub中Windows为第一启动项, 而且我觉得grub加了主题之后真的非常漂亮

# Windows/Ubuntu时间不同步
处理完之后基本就完成了

# Ubuntu 16.04升级到18.04
## 升级出现的问题
之前在公司使用的是vmware 10.0 + Ubuntu 16.04LTS, 本来是想要重新安装一个Ubuntu 18.04的, 但是安装完成之后发现一直安装不了vmware-tool, 需要安装ifconfig, 但是ifconfig又有依赖包安装不了, 但是之前由16.04升级18.04也还是不成功, 先是什么python版本不对, 然后又需要更换到原始源, 更换之后又老出现连接不上一个ip地址导致升级失败
## 解决方案
打开Software&Updates, 然后进入Settings->Ubuntu software, 把能选的都选上,download from选择main server, 然后进入到Updates, 重要安全更新和推荐更新, 回到终端执行`sudo apt update;sudo apt dist-upgrade`, 更新时间会非常长, 建议是晚上前更新


