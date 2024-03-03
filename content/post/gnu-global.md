---
title: "工具盒子-GNU global"
date: 2014-11-15T21:04:23+08:00
tags: [
	"linux",
	"tools",
]
categories: [
	"cheatsheet",
]
---

### gnu global 
源码标记，浏览源码挺好用的工具,轻量级的，简单易用。 gtags类似ctags,但是效率比ctags高，具体比较查看[这里](https://github.com/OpenGrok/OpenGrok/wiki/Comparison-with-Similar-Tools)(OpenGrok使用相对比较复杂)，而且未来支持的语言也比较多。对Linux-2.6.32源码390M的文件进行标签产出289MB的标签文件。  

可以通过源码安装就OK <code>wget http://tamacom.com/global/global-6.3.2.tar.gz; tar zxvf global-6.3.2.tar.gz; cd global-6.3.2; ./configure; make && make install</code>

在需要查找的目录下运行<code>gtags</code>,会生成三个文件： GTAGS--定义的函数变量； GRTAGS--引用的函数变量； GPATH--函数变量所在文件的路径。 

<!--more-->
  
有用的命令：  
1. <code>global -f DIR/fileC.c --color</code>这个命令看这个文件下有哪些函数，比较有用,加-l在当前目录下查找  
2. <code>global -r func</code>函数在哪些文件中引用  
3. <code>global -ax func</code>函数所在的目录文件具体哪些行  
4. <code>global -P file</code>匹配文件目录文件，类似<code>find ./ -type d | grep file</code>  
5. <code>global -g func</code>匹配有func字符的文件，和grep功能一样  
6. 如果代码跟新了，需要更新标签文件，<code>global -vu</code>  

通过globash进入global命令模式，其中<code>show -e/v/l/g Nth</code>分别用emacs,vi,less,mozilla 来浏览第N个文件标记所搜得关键词的内容，或者不指定用编辑器，采用环境变量EDITOR定义号的编辑器来查看(在.bashrc中添加<code>export EDITOR=vim</code>)，个人比较喜欢用vim；使用tags查看已经查看过的tags；使用mark来标记查询过的标签tag,<code>mark -l</code>列出mark列表，<code>mark Nth</code>通过默认编辑器浏览mark的第N个文件；<code>pop</code>对记录过的查找标签命令的结果栈退出栈顶结果，并显示处理后的栈顶结果；<code>cookie</code>用来记录当前的文件夹，<code>cookie -l</code>查看cookie记录的文件夹，<code>warp Nth</code>返回列出的第N个文件夹。  
其他简单的命令可以通过ghelp查看。  

还有个htags是用来生成超文本文件的，方便在浏览器下查看，如果在命令行的开发坏境下，这个有个毛用哦~

使用gtags-cscope进入client模式具体操作按？帮助文档操作，(哈哈，貌似像个搜索引擎，tag的缓存文件采用的b-tree)。具体见[https://www.gnu.org/software/global/globaldoc_toc.html#Requesting-the-initial-search](https://www.gnu.org/software/global/globaldoc_toc.html#Requesting-the-initial-search)

gtags,global其他具体命令，参考帮助文档或者查看[Tutorial](https://www.gnu.org/software/global/globaldoc_toc.html)

gtags可以作为ctags的扩展进行使用，具体参考[https://www.gnu.org/software/global/globaldoc_toc.html#Plug_002din](https://www.gnu.org/software/global/globaldoc_toc.html#Plug_002din	)

### VIM中使用global

可以之间从安装好的global文件间share中将gtags.vim复制到~/.vim/plugin文件夹中：<code>cp /usr/local/share/gtags/gtags.vim $HOME/.vim/plugin</code>  

或者如果使用vundle来管理的话，BundleSearch gtags 安装；或者直接<code>cd $HOME/.vim/bundle && git clone https://github.com/vim-scripts/gtags.vim gtags</code>，或者用git submodule来管理，原理都差不多，自己vim以前用vundle管理过，就直接将自己用的插件全部整合在一起了，去掉vundle插件，采用git submodule管理插件模块，但是vundle整体管理框架还在，[vim配置](https://github.com/weedge/Vim-PHP-IDE)。  

在vim的命令模式下采用Gtags命令，参数和shell命令行一样，查找出来的list列表在quickfix window 中展现，操作的话与grep命令差不多，但是筛选比较细些，用-g参数就和grep一样，后面可以加上grep一样的参数。

具体参考[https://www.gnu.org/software/global/globaldoc_toc.html#Vim-editor](https://www.gnu.org/software/global/globaldoc_toc.html#Vim-editor)

### 总结
global作为一个轻量级的源码标记，浏览源码挺好用的工具，总体体验了吧，gtags生成的文件进行了细分，tag的检索效率相对提高了，如果源码比较大型的话，采用global -vu 来更新tag文件，要快很多；用vim进行开发，可以使用gtags.vim这个插件，与grep配合使用吧(当所检索的函数/变量tag比较模糊的时候，可以先用grep查询，在使用Gtags定位).好用程度，只有继续体验一段时间吧。。。
	
