---
title: Android P wpa_cli启动
date: 2019/08/27
tags:
  - android
  - wifi
categories:
  - android
  - wifi
abbrlink: 8091938f
---

# 前提
1. 需要root手机
2. 只针对Android9.0的MTK平台

# 启动步骤
1. 查看wpa_supplicant是否启动`ps -A | grep wpa`, 如果有类似下面的信息打印则说明启动成功了, 否则需要先启动wpa_supplicant, 一种简单的方式是打开wifi就行了, 或者是使用命令行启动
```shell
# wpa_supplicant进程信息
wifi          1364     1   13208   8664 poll_schedule_timeout 0 S wpa_supplicant

# 启动wpa_suplicant进程命令
#Android O
/vendor/bin/hw/wpa_supplicant -d -B –iwlan0 –Dnl80211 -c/data/misc/wifi/wpa_supplicant.conf -C/data/misc/wifi/sockets
#Android P
/vendor/bin/hw/wpa_supplicant -d -B –iwlan0 –Dnl80211 -c/data/vendor/wifi/wpa/wpa_supplicant.conf -C/data/vendor/wifi/wpa/sockets
```
2. 启动wpa_cli, 在任意目录下执行`wpa_cli`就行了

# 问题
最常见的就是`Could not connect to wpa_supplicant: wlan0 - re-trying`, 首先是查看`/data/vendor/wifi/wpa/sockets`是否有wlan0文件, 如果没有则说明`ctrl_interface`设置的不正确, 回退到`/data/vendor/wifi/wpa`目录下,pull出`wpa_supplicant.conf`, 文件内容大致如下
```shell
ctrl_interface=wlan0
update_config=1
manufacturer=MediaTek Inc.
device_name=Wireless Client
model_name=MTK Wireless Model
model_number=1.0
serial_number=2.0
device_type=10-0050F204-5
os_version=01020300
config_methods=display push_button keypad
p2p_no_group_iface=1
driver_param=use_p2p_group_interface=1
hs20=1
```
修改`ctrl_interface=/data/vendor/wifi/wpa/sockets`, **在/vendor/etc/wifi/目录下也有一个wpa_supplicant.conf文件, 如果/data/vendor/wifi/wpa/中的没有生效的话可以尝试这个文件**, 修改之后重启之后再执行`wpa_cli`, 如果还是不行可以使用`strace wpa_cli -i wlan0`进行调试

参考文章:
[Android Oreo8.0 使用wpa_supplicant和wpa_cli(更新AndroidPie9.0)](https://blog.csdn.net/andytian1991/article/details/80486835)
[Why is wpa_cli producing error “Could not connect to wpa_supplicant: wlan0 - re-trying”?](https://raspberrypi.stackexchange.com/questions/63399/why-is-wpa-cli-producing-error-could-not-connect-to-wpa-supplicant-wlan0-re)
