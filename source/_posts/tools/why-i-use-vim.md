---
title: 我为什么要使用VIM
date: 2019/07/24
tags: tools
categories: tools
abbrlink: 77c23cfe
---
# 前言 
作为一名Android应用工程师, 个人算是用的比较多的编辑器了, 从IDE Eclipse, AndroidStudio到编辑器SublimeText, UltraEdit, gedit, vscode, atom这些我都用过, 虽然都是浅尝辄止, 但是从Android开发上来说, AndroidStudio无疑是最合适的, 丰富的插件和功能, 尤其是Live Template功能, 自定义起来不要太爽, 唯一的缺点就是内存消耗太大了, 没有16G内存+SSD感觉可以不用尝试了, 编辑器看起来都差不太多, UltraEdit我一般用来搜索LOG比较方便, SublimeText无疑是各方面非常优秀的编辑器, 但是要收费, vscode, atom这两个优点在于插件多, 但是消耗起内存一点都不比Android Studio差.

# 使用VIM的原因
逼迫我使用VIM唯一原因就是--内存太小, 虽然公司给配了16G的内存, 但我是Android系统应用开发工程师啊, 这个Ubuntu虚拟机必须时刻都给开着, 为了让主机能有开多一点应用, 只能给虚拟机分配6G内存, 加上Android加速编译的Java进程, 给的内存根本不够用, 而且公司的主板只有两个内存插槽, 想扩展都没办法. 所以这种情况下就只能使用VIM这种耗内存比较小的应用了, 相较于vscode动不动上100MB, atom 300多MB的应用(根据你打开的文件个数和大小), VIM的平均6,7MB的在内存方面就是碾压.

# 使用技巧
用VIM开发Android应用其实是相当困难的, 有多次都想要放弃了, 但都在用其他编辑器需要每天重启4-5次虚拟机的压力下坚持下来了, 距今已经有一年多了. 这里有些关于VIM开发Android的小技巧可以分享一下

## 插件

```vimscript
Plugin 'VundleVim/Vundle.vim'
Plugin 'scrooloose/nerdtree'
Plugin 'Yggdroot/indentLine'
Plugin 'junegunn/fzf.vim'
Plugin 'mileszs/ack.vim'
Plugin 'jiangmiao/auto-pairs'
Plugin 'aklt/plantuml-syntax'
Plugin 'vim-airline/vim-airline'
Plugin 'vim-airline/vim-airline-themes'
```
上面就是我目前最常用的配置, fzf插件用不了的可以使用`Plugin 'Yggdroot/LeaderF'`, 下面说一下各个插件的作用:
1. Vundle就是用来安装插件, 这个没啥好说的, 按照[github](https://github.com/VundleVim/Vundle.vim)进行安装, 有了它, 才能使用`Plugin 'Plugin address'`这种方式安装插件, 对应的还有bundle也是一样的.
2. nerdtree是用来显示vim打开的目录的, 我一般最常用的就是`map <F3> :NERDTreeFind<CR>`, 用`F3`快捷键找到当前打开的文件的目录, 查看目录结构, 其实可用度一般, 不知道为什么网上那么多人推崇, 可能是我不太会用
3. indentLine就是显示缩进的插件, 这样就不用数空格了, 对于看代码来说还是很不错的
4. fzf文件搜索神器, 比find要快多了, 就是下载要翻墙, 当然也可以用LeaderF, 只是这个只有Vim插件, fzf可以针对终端搜索, 最常用的操作就是`nnoremap <C-T> :FZF<CR>`, 用Ctrl+T的方式进行搜索文件, 强烈推荐
5. ack代码搜索神器, 基本上看源码是必备的, 当然也可以使用ag进行搜索`let g:ackprg = 'ag --nogroup --nocolor --column'`, 这样也不错, 就是不知道为啥使用ag的时候没法加参数
6. auto-pairs, 自动匹配, 当你输入`(`会自动变成`()`, 删除也是同理, 对于双引号, 单引号, 括号这些输入那是相当的舒服
7. plantuml-syntax, 这个是使用plantuml的插件, plantuml是一款非常非常好用的uml工具, 用起来会上瘾, 它是将画uml的方式用代码写出来, 强烈推荐这款工具, plantuml-syntax就是将其语法高亮
8. vim-airline和vim-airline-themes这两个插件就是让你的tab显示的更加漂亮

## 查看Android源码
对于不管是不是Android系统开发工程师来说, 阅读Android frameworks层的源码那都是必不可少的, 建议有条件的可以搭建一个OpenGrok, 搜索起来不要太快, 回到正题, 那么怎么用vim看Android源码呢
### 打开文件
我打开文件一般都是先切换到android根目录下面先`source build/envsetup;lunch`一下, 新建一个终端tab, 然后进入到对应目录, 比如: `cd package/apps/Settings;vim .`, 这样有几个好处
1. 一个终端tab只用做编译代码, 其他的tab用来看代码. 
2. 进入到Settings目录下面可以方便搜索代码和文件, 因为fzf和ag都是在当前目录下搜索, 所以如果是查看Settings目录下, 进入这个目录刚刚好

如果是需要看frameworks的代码的话, 最好是进入frameworks的下一层目录例如`frameworks/base`, 因为frameworks下面的模块也是相当多的

<iframe height=450 width=1000 src="open-dir.gif" frameborder=0 allowfullscreen></iframe>

### 搜索文件
我一般使用的都是fzf进行搜索文件, 终端也可以用, 一般都是设置快捷键`nnoremap <C-T> :FZF<CR>`, 使用Ctrl+T打开搜索界面, 搜索完之后使用Ctrl+T新建的tab中打开文件, 用快捷键gt或者是gT, 进行切换tab

<iframe height=450 width=1000 src="search-file.gif" frameborder=0 allowfullscreen></iframe>

### 搜索源码
使用Ack的时候, 如果设置了`let g:ackprg = 'ag --nogroup --nocolor --column'`, 可以直接使用`:Ag`,搜完之后使用Ctrl+T在新建的tab中打开文件, 如果没有设置, 则要使用`:Ack!`, 不过Ack搜索是放在Quickfix中

<iframe height=450 width=1000 src="ag-search.gif" frameborder=0 allowfullscreen></iframe>

### 跳转
一般文件内部跳转源码我都是都是`/`搜索的方式进行搜索, 或者是gD快捷键进行搜索, gD其实是go to define的意思, 但是这个比较蠢, 只会跳转到第一次出现的地方. 不过也有一些小技巧, 比如搜索内部类的时候可以带上class前缀, 搜索方法的时候带上返回值, 搜索属性定义或者是类导入的时候带上分号, 搜索赋值或初始化的时候带上空格+等号, 其实跳转也不是很慢, 当然文件外部跳转还是使用Ack
还有翻页, 我一般都是Ctrl+D进行半页半页的翻, 然后会设置`set mouse=n`, 让normal模式可以也可以滑动鼠标, 并且也可以将光标精准的放到对应的位置, gg可以跳到文件开头,G可以跳转到文件尾, nG可以跳转到对应的行
##### 查看LOG
在接触到vimgrep之前几乎没法看Android LOG, 因为使用`/`搜索没法预览, 只能一个一个跳转, Ack并不能针对当前打开的文件进行搜索, 只能手动输入文件名, 后来看了一篇关于vimgrep搜索的文章, 果断是把UltraEdit给卸载了.
`:lv search_word %`可以搜索到当前打开文件下所有*search_word*的单词, 并显示个数, 然后使用`:lw`就可以将搜索到的行输入到Quickfix中, 还可以设置快捷键`nnoremap <leader>s :lv /<c-r>=expand("<cword>")<cr>/ %<cr>:lw<cr>`这样就可以直接搜索光标所在单词了

后来找到一种新的方式, 因为"%保存了当前打开文件文件的相对路径, 所以直接用`:! ag search_word %`命令就可以获取了, 

## 写代码
编辑器没有自动提示写代码确实很头疼, 我有几次也很想放弃了, 有一些java语言自动完成的插件, 但是都不怎么符合要求, 下面有一些小技巧可以加快一些写代码的速度
### 复制和粘贴
都知道dd,yy快捷键是剪切和复制, 用p进行粘贴, 但是复制到系统剪贴板的快捷键"+实在是太反人类了, 所以我一般都是将其映射到一个快捷键`nnoremap <space>y "+`, 映射的时候得注意习惯, 因为复制的次数太多的话, 这个习惯就没法改了, 现在已经变成神经自动反应了,需要复制的先打Space+y
### 自动补全
这个真的能提高很多写代码的效率, 快捷键Ctrl-N可以自动补全曾经出现的字符串, 跟IDE的自动补全非常相似, Ctrl-N选择下一个, Ctrl-P选择上一个, 而且这个是可以找到你打开的目录下所有的文件中出现过的字符串, 这个跟IDE的补全几乎就是一样的, 除了不能自动补全SDK下的代码, 况且作为系统应用, 很多隐藏类和隐藏方法也不能进行补全

<iframe height=450 width=1000 src="autocomplete.gif" frameborder=0 allowfullscreen></iframe>

### 善用abbreviation
这个是类似于Android Studio里面的Live Templates, 当然是无法像LiveTemplates那么功能那么多, 但也足够方便了. 下面我的一些例子
```vimscript
iabbrev syso System.out.println("");<ESC>2hi "添加<ESC>2hi的目的是让光标停留在双引号中间
iabbrev logd Log.d(TAG, "");<Left><Left><Left>
iabbrev logi Log.i(TAG, "");<ESC>2hi
iabbrev consti public static final int
iabbrev constS public static final String
iabbrev docj /**<cr><cr>/<ESC>kA
```
使用这个iabbrev的方法很简单, 保证是在插入模式, 输入`logd`然后再输入空格或者是Ctrl+]就可以变成了`Log.d(TAG, "");`.

<iframe height=450 width=1000 src="iabbrev.gif" frameborder=0 allowfullscreen></iframe>

abbreviation的设计的目的用一句话简单概括就是:**代替你的手**, 所以你在设计abbreviation的时候**只需要将你想要操作的键一个个的输入到abbreviation里面**, 一个abbrev形成了, 如果你对shell足够熟悉, 几乎可以将AndroidStudio的LiveTemplates完全添加进去

## 另外的一些小技巧
通过以上的小技巧可以应付大部分Android系统应用开发的问题, 有些个人使用的一些小技巧, 主观性比较强:
1. 双击双引号可以为一个单词添加双引号,`nnoremap "" viw<esc>a"<esc>hbi"<esc>lel"` 对于String的用的比较多
2. 用Ctrl+J代替ESC键`inoremap <C-J> <esc>`, 因为ESC键实在是离我太远了
3. 用H代替0回到行首`nnoremap H 0`, 用L代替$到行尾`nnoremap L $`
4. 暂停vim回到shell的时候保存所有的文件, `nnoremap <C-Z> :wa<CR><C-Z>`, 在shell中使用fg可以回到vim
5. 在tab比较多的时候可以先用`:tabs`查看tab列表, 然后用`ngt`直接可以切换到对应tab, 这种在tab一行显示不下的时候用起来比较方便, 为啥vim不能多行显示tab呢
