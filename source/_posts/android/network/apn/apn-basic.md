---
title: APN基础
date: 2019/07/22
tags:
  - apn
categories:
  - android
  - network
abbrlink: ffeba2ab
---
本文将会分为三个部分进行讲解APN, 分别是apns-conf.xml, ApnSettings, TelephonyProvider，这三个是属于上层配置会涉及到的点。

# 基础概念
1. APN: Access Point Name, 是决定了上网类型，由Modem在注网的过程中根据Sim卡的信息从TelephonyProvider中进行获取，并注册上对应的网络
2. SPN: Service Provider Name, 通常指的是注册的网络的运营商的名称
3. MVNO: Mobile Virtual Network Operator, 移动虚拟网络运营商，对应的是移动网络运营商(MNO)，MNO和MVNO两个的mccmnc都是一样，通过imsi,gid1,spn,pnn这些类型进行匹配来与MNO进行区分
4. MNO: Mobile Network Operator, 移动网络运营商,口头上用实体运营商和虚拟运营商区分MNO和MVNO, SIM卡的运营商, 举个例子: 中国移动是实体运营商,但是它还有一个叫蜗牛移动的虚拟运营商
5. IMSI: International Mobile Subscriber Identity(国际移动用户识别码), 里面包含了一个mccmnc码和MSIN码
6. PLMN: Public Land Mobile Network(公共陆地移动网络), 就是指的是mccmnc
7. MCC: Mobile Country Code, 用于指代一个国家的移动码, 每个国家基本就是一个,一般是3位,除了中国和印度,中国大陆是460, 台湾地区是466, 印度是404和405
8. MNC: Mobile Network Code, 由运营商定义, 2-3位,由不同地区决定
9. MMS: Multimedia Messaging Service, 俗称彩信

# apns-conf.xml
### 属性讲解
apns-conf.xml主要涉及的一些属性类型, 必须以<apn/>作为标签,
例如:
```xml
<apn carrier="移动互联网"
     mcc="460"
     mnc="00"
     apn="cmnet"
     type="default,supl"
     protocol="IPV4V6"
/>
```
1. carrier: 运营商名称, 显示在APN列表上的名称, 无需太关注
2. mcc: 用于识别用户所在国家
3. mnc: 用于识别国家的运营商
4. apn: access point name(接入点名称), 由运营商给出
5. type: apn类型, 一般有default, dun, supl, mms, ims, xcap, hos
6. protocol: 注网时使用的协议, 一般是IPV4V6, IP, IPV6, 默认是IP
7. roaming_protocol: 漫游时注网时的协议, 类型同protocol
8. mmsc: mms专用, mms的服务器
9. mmsproxy: mms专用, mms的代理服务器
10. mmsport: mms专用, mms的端口
11. user: 用户名
12. password: 密码
13. authtype: 认证协议, 0: None, 1: PAP, 2: CHAP, 3: PAP or CHAP
14. mvno_type: 虚拟运营商类型, 一般为: imsi, spn, gid, pnn(这种类型还没见过), 用于识别是不是mvno
15. mvno_match_data: 虚拟运营商的值, 一般由运营商给出, 根据mvno_type设置 
16. user_visible: 是否显示在ApnSettings UI当中, 0或者false为不显示, 1或者true为显示, 默认是显示
17. user_editable: 是否在ApnEditor UI中可编辑, 0或者false为不可编辑, 1或者true为可编辑, 默认是可编辑<br>

### 注意事项
1. mcc, mnc, apn, type这四项是必选的, 其他都是可选项
2. authtype在不设置user和password的情况下默认0, 但是设置了的情况下默认是3
3. ims类型的主要是针对ims网络,主要是用于vowifi和volte使用
4. xcap类型主要是针对ss业务, 一般是呼叫转发, 呼叫等待, 限制呼叫, 隐藏号码的功能<br>

# TelephonyProvider
TelephonyProvider主要是两个表,carriers表主要是存储APN属性, 也就是apns-conf.xml中获取的属性; siminfo则主要是存储sim卡的属性. 
TelephonyProvider的代码较多,这里只选取的是Apn加载的流程.对应的代码在packages/providers/TelephonyProvider/src/com/android/providers/telephony/TelephonyProvider.java
### APN加载
1. 从`initDatabase()`开始
2. 接下来通过`getApnConfFile()`获取系统中的apns-conf.xml, 一般是在/system/etc目录下
3. 通过`XmlPullParser`进行解析xml, 然后通过`loadApns()`以及`getRow()`方法将解析的属性放到`ContentValues`当中
4. 最后通过`insertAddingDefaults()`插入到数据库当中

### 注意事项
1. TelephonyProvider的数据库在第一次开机的时候就创建了, 以后重启包括OTA升级都不会再重新创建了, 只能通过删除数据库或者是重置设备来重新创建
2. 调试技巧,可以将手机root, 然后进入/data/user_de/0/com.android.providers.telephony/databases, 将telephony.db删除, 重启手机就可以
3. 重复APN合并, APN的一些属性具有唯一性, 具体看`CARRIERS_UNIQUE_FIELDS`列表中的属性, 这些唯一性的属性, 在通过`insertWithOnConflict`方法进行添加的时候, 如果这些属性**全部相同**则会出现冲突的情况, 这个时候需要合并两个APN, 通过`mergeFieldsAndUpdateDb`进行合并, 只有type和mask值合并
   3.1 针对type的合并, 主要是将新APN中type类型通过逗号进行区分, 然后新加的类型合并到已插入的APN中
   3.2 针对bear_bitmask和network_type_bitmask的合并,则是两个值的合并
4. 默认APN会单独放在SharedPreference中存储, 一般是由frameworks层进行创建, 由ApnSettings进行显示
<br>

# ApnSettings
ApnSettings主要是APN的UI界面显示, 涉及到 `ApnSettings.java`和`ApnEditor.java`两个文件, 分别用于APN显示和APN编辑, APN的显示和编辑的逻辑比较简单, 显示的逻辑主要是在`fillList()`方法中, 编辑主要是`validateAndSaveApnData`更新数据库
### 注意事项
1. ims类型apn可以通过属性控制显示或者不显示`where.append(" AND NOT (type='ims')");`
2. mms类型apn不可以设置为默认APN, 即不可以选择
3. mvno类型的APN会优先匹配显示, 只有当mvno类型APN不存在时, 才会显示对应mno类型的APN, 这样会有两种情况, 需要酌情判断
   3.1 插入的sim卡真的是mno的卡
   3.2 插入的sim卡是mvno的卡, 但是这张卡的mvno类型的apn没有配置, 此时就需要向FAE获取mvno的apn, 并配置到apns-conf.xml中
4. MTK平台中会有默认APN的选择, 这时要注意排序的问题, 因为是按照第一个APN设置为默认APN, 最好按照id(MTK是按照名称排序, 有点坑)进行排序, 这样可以手动调整apns-conf.xml中apn的顺序来选择合适的默认的apn
