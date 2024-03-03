---
author: "weedge"
title: "像黑客一样写博客"
date: 2013-01-01T20:26:23+08:00
tags: [
	"octopress",
	"git",
]
categories: [
	"blog",
]
---
## 背景

最近一段时间想整个博客，来记录一些事情，感觉记忆力不如从前了，想起以前有位老程序员，身边总是挂一个小挎包，装了一些小便条(todo list)；我想人的精力是有限的，当做完一件事， 过段时间，可能忘了曾今做过的事，说的话；本来想用worldpress，但是貌似需要申请空间，不同以往了，现在都是收费的了，无意间发现了[octopress](http://octopress.org/docs/),这是个好东东啊，轻量级的，不像worldpress臃肿，不需要使用mysql,通过git直接可以部署到github pages上搭建一个副本。也可以部署到[heroku](http://www.heroku.com/)；国内好像也有对octopress的部署支持。

<!--more-->
## octopress

在开始写过程之前先说下为毛这样做，其实也是觉得对新鲜的东东比较感兴趣吧！

1. 用octopress写博客，就像网上说的-像黑客一样写博客，貌似需要那么一点伪技术，看上去就像geek那样啦~~ 呵呵。

2. 整个服务的搭建是完全免费的，也许对于我这个比较穷困专业挨踢男来说是一个开源节流的渠道吧。

3. 使用octopress来写博客，支持[markdown](http://wowubuntu.com/markdown/basic.html),和[haml](http://en.wikipedia.org/wiki/Haml),
个人喜欢使用前者，github上可以直接对其编辑；而且主要的是用命令来发布，这动作很cool,like it~!,感觉比较碉，哎呦，不错哦~！
可以使用`rake -T`来查看信息,发布常用的命令:
`rake install[theme]  rake preview  rake new_post[title]  rake generate  rake deploy`

4. 最后是可以离线写博客还是挺爽的，用git对文章进行备份，以及静态页面生成，style也不错。
	
    这里不具体的介绍octpress搭建在gitHub的过程，可以查看octpress的在线文档，说的很清楚的。
不过在deploy的时候可能需要ssh key,在终端执行：
`[[ -f ~/.ssh/id_rsa.pub ]] || ssh-keygen -t rsa`
将其显示:
`[[ -f ~/.ssh/id_rsa.pub ]] && cat ~/.ssh/id_rsa.pub`
    然后进入[https://github.com/account/ssh](https://github.com/account/ssh)中点击Add another public key，粘贴前面步骤复制的信息，请格外注意，不要在Title中填写内容，直接将复制的内容粘贴到Key中，然后点击：Add Key即可。
	还可以参考：[github上的帮助文档](https://help.github.com/articles/generating-ssh-keys)

<!--more-->

由于自己对ruby不熟悉，但是想使用tag插件，不想使用octpress自带的分类插件，所以在网上找了下，采用了[https://github.com/jsw0528/octopress/tree/mrzhang_me/](https://github.com/jsw0528/octopress/tree/mrzhang_me/)这个风格，与原始版本的的区别如下： 

1. [.Rvmrc](https://github.com/jsw0528/octopress/blob/mrzhang_me/.rvmrc)  
神仙同学用的是Ruby1.9.3，Octopress最低要求1.9.2(采用rvm安装ruby,比较方便)

2. [Gemfile](https://github.com/jsw0528/octopress/blob/mrzhang_me/Gemfile)  
源改成了ruby.taobao.org的，去掉了部分gem的版本号限制

3. [Rakefile](https://github.com/jsw0528/octopress/blob/mrzhang_me/Rakefile)  
这个文件可以修改自己喜欢的编辑方式(haml,markdown)，octopress默认使用markdown,作者采用haml,
修改了时区，降低主题对custom目录的依赖

4. [_config.yml](https://github.com/jsw0528/octopress/blob/mrzhang_me/_config.yml)  
定制化配置信息，日期格式、永久链接、微博，评论(默认支持disqus,可以使用国内的多说)等等

5. [Plugins/sh_js.rb](https://github.com/jsw0528/octopress/blob/mrzhang_me/plugins/sh_js.rb)  
代码高亮插件(plugins是用ruby写的插件库，自己可以扩展)

6. [Plugins/tag_generator.rb](https://github.com/jsw0528/octopress/blob/mrzhang_me/plugins/tag_generator.rb)  
支持中文的Tag插件

7. [.Themes/blog/](https://github.com/jsw0528/octopress/blob/mrzhang_me/.themes/blog/)  
神仙同学的博客主题(./themes主题风格文件夹，可以自己定制)

## 总结

在看source(来自第7条所说文件同rake install[theme]复制生成的)源码时，发现采用缓存([Cache Manifest](http://www.cnblogs.com/brainmao/archive/2011/09/27/2193495.html))，
主要在页面中通过 iframe 来引用offline-cache.html这个静态文件，其中assets.manifest缓存了样式，图片，js信息，
以达到我们不缓存当面页面，只缓存我们希望缓存文件的目的，所以有时候你对其他页面修改了，
比如修改该了post的文件名的话，会以为没有更新成功的情况，其实已经在github服务器上了，要做的就是清除浏览器中的页面缓存或者按Ctrl+F5刷新下。

ok~ 开始像黑客一样写博客吧~！

## 参考
- [octopress-黑客一样写博客](http://www.yangzhiping.com/tech/octopress.html)
- [Markdown写作浅谈](http://www.yangzhiping.com/tech/r-markdown-knitr.html)
- [服务器端Octopress搭建及移动方案](http://lucifr.com/2011/12/21/octopress-on-server-and-portable-scheme/)
