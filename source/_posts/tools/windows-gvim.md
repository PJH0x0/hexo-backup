---
title: Windows下使用gvim的总结
date: 2019/09/17
categories: tools
tags: tools
abbrlink: d7e493ba
---
*系统版本: Windows7*
*VIM版本: 8.1*
# 安装
直接下载第一个*gvim81.exe(ftp)*就行, [下载地址](https://www.vim.org/download.php#pc)
# 修改_vimrc
Windows下的GVIM是安装在`C:\Program Files (x86)`目录下, 配置文件是在`C:\Program Files (x86)\Vim`目录下, 但是要修改_vimrc文件必须要要有写入权限.
修改当前用户使其拥有文件的完全控制权限:右键_vimrc->安全->编辑->选中当前用户->选择完全控制->确认后退出
# 删除备份文件
每次用VIM打开文件都会有两个备份文件, 删除的方式是进入到vimrc_example.vim文件中注释掉备份的代码
```vimscript
if has("vms")
  set nobackup		" do not keep a backup file, use versions instead
"else
"  set backup		" keep a backup file (restore to previous version)
"  if has('persistent_undo')
"    set undofile	" keep an undo file (undo changes after closing)
"  endif
endif
```
vimrc_example.vim在`C:\Program Files (x86)\Vim\vim81`目录下, 当然也需要修改vimrc_example.vim的权限使其可以让当前用户写入
# 设置主题
标题栏中只能修改当前窗口下的主题和字体, 要修改全局的还是要编辑_vimrc,加入以下两行代码
```vimrscript
colo evening
set guifont=Consolas:h10:cANSI
```
不过windows下的vim似乎无法打开文件夹, 只能当做是UltraEdit的替代品了
