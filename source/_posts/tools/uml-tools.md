---
title: UML工具plantuml推荐
date: 2019/7/29
categories:
  - tools
tags:
  - tools
abbrlink: 9b805327
---

# 原因
其实以前也用过一些uml画图工具, 但是总是画的比较慢, 可能比较手残的关系吧. 导致我一直不太愿意画uml图. 直到用了plantuml之后, 分析源码不画个时序图, 手就开始痒了, 实在是太方便了.
plantuml有以下几点优势:
1. 速度快, 因为plantuml是代码实现uml图, 所以再也不用挪方块了, 直接输入代码就可以了
2. 支持插件形式, plantuml可以完美的以插件的方式集成到各个IDE中, Android Studio, Eclipse等等
3. 完全免费, 也没有任何付费的高级会员

# 使用
## 下载
plantuml支持在线服务, 输入完之后, 通过右键->图片另存为可以保存生成的uml图, 也挺方便的. 或者是通过IDE插件进行集成, 但是我都是通过命令行生成uml图, 因为我一般都用vim进行开发.
1. 下载jar包, 这里是[下载地址](http://plantuml.com/zh/download), 下载完之后最好放在一个固定的地方, 我一般放在home目录下`mkdir ~/.plantuml;cp plantuml.jar ~/.plantuml` 
2. 下载graphviz, Ubuntu直接用`sudo apt-get install graphviz`, 不是Ubuntu的系统就需要到[官网](https://www.graphviz.org/download/)下载一下

## 语法
plantuml各个uml图的语法不一样, 但是都是`@startuml`开始, `@enduml`结束, 具体的语法可以在网页的topbar中寻找.

## 生成uml图片的命令
通过`java -jar ~/.plantuml/plantuml.jar sequenceDiagram.puml`进行生成png图片,速度还可以吧, 我这个命令和官网里面的不太一样`java -jar plantuml.jar sequenceDiagram.txt`, 使用完整的路径的原因是因为我没加环境变量, 使用.puml的后缀是为了vim的语法高亮插件

## 使用主题
planuml是可以添加主题的, 可以改变uml图线条以及字体颜色, 我用的是一个日本工程师的主题[cf_theme](https://github.com/go-zen-chu/plantuml_cf_theme), 下载里面的config.txt, 然后也放到.plantuml目录中, 然后生成uml图片的命令也需要改一下`alias plantuml='java -jar ~/.plantuml/plantuml.jar -config ~/.plantuml/configs/cf_theme'`, 这里注明一下我是将config.txt改成了cf_theme文件, 放到了`~/.plantuml/configs/`目录下
使用了alias之后我们的命令就可以变成`plantuml sequenceDiagram.puml`, 完美

## 语法高亮
如果作为IDE插件使用直接就有语法高亮, 但是vim还需要一个插件`Plugin 'aklt/plantuml-syntax'`, 并且这个插件识别语法需要你的文件是.puml文件后缀
