---
title: PkMS初次总结
date: 2019/12/30
categories:
  - android
  - framework
tags:
  - android
  - PackageManagerService
abbrlink: ff6a3f04
---

# 开机扫描流程
PackageManagerService.java:
PackageManagerService() -> scanDirTracedLI() -> scanDirLI() -> scanPackageChildLI() -> addForInitLI() -> scanPackageNewLI() -> scanPackageOnlyLI() -> commitReconciledScanResultLocked() -> commitPackageSettings()
解析APK的流程:
scanDirLI() -> ParallelPackageParser.submit() -> PackageParser.parsePackage() -> parseClusterPackage()/parseMonolithicPackage() -> parseClusterPackageLite()/parseMonolithicPackageLite() -> parseApkLite() -> parseApkLiteInner()
解析之后通过ParallelPackageParser.take()获取ParseResult, 之后获取Package数据结构, 对其进行一系列处理

# PackageManagerService内部主要的数据存储
1. Package, Apk解析之后生成的数据结构,基本是最主要的数据结构
2. mPackages, 里面是解析之后的数据结构Package的集合, 放在PackageMangerService中保存
3. ComponentResolver, 是解析之后Package中AndroidManifest中四大组件中的集合

# APK解析简单说明
主要是通过ApkAssets进行解析,通过native层进行解压缩.apk文件,然后获取里面对应的目录和文件进行解析

# APK签名
从collectCertificates方法开始,解析出SigningDetails数据结构结束, 详情可参考[APK签名机制原理详解](https://www.jianshu.com/p/286d2b372334)

# APK install简单流程
PackageInstaller->InstallInstalling.java
onCreate()(构建SessionParams, 并调用createSession()) -> onResume()(触发线程) -> doInBackground()(调用openWrite进行拷贝文件) -> doPostExcute()(调用session.commit()方法)

PackageInstallerSession.java
openWrite() : 将文件拷贝至/data/app/vmd-uid.tmp目录下
commit() -> mHandlerCallback.handleMessage() -> handleCommit -> commitNonStagedLocked() -> 


