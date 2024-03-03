---
author: "weedge"
title: "更换博客"
date: 2017-10-12T10:26:23+08:00
tags: [
	"hugo",
	"git"
]
categories: [
	"blog",
]
---

&emsp;&emsp;博客有段时间一直没有跟新过了，说白了，就是太懒了，或者没有动力去push你去干好这件事情；看着以前的博客，寥寥无几的几篇，质量也不高；早上上班经常看一些公众号文章，有个作者每天坚持写一篇文章，都已经坚持了大半年了，从生活的思考记录，到技术的积累，而且输出的文章质量不错，至少自己读了之后会产生一些共鸣，或者学到一些知识点。

> 经常听到技术人总结的话：技术是一个积累的过程，从别人那里看到的，和自己去动手实现的是两回事，别人趟过的坑，你再重新踩一次，也许会遇到新的坑，这些踩过之后，把这些知识点和满坑方案记录下来，日积月累，是对以后是有帮助的。

<!--more-->
&emsp;&emsp;以前的博客是用[octopress](http://octopress.org/docs/)来生成博客的静态文件的，最近发现这个没有更新了; 自己在想是不是可以自己按照这个思路生成一套，加上最近在学习golang；在网上索搜了用golang生成博客的工具，确实有个不错的开源工具[hugo](https://gohugo.io/)，使用后，发现超级方便，生成到发布就两步，相比繁琐操作的octopress,要简单方便的多，而且生成的前端模板用的是[mustache](https://mustache.github.io/)，模板语法在前端比较常用; 

1. macOS下安装，直接可以通过brew来安装`sudo brew install hugo`就可以使用了；

2. 建立自己的博客，用markdown写文章，然后本地查看，检查ok在发布到github page上；具体操作看[官方文档](https://gohugo.io/documentation/)吧， 这里就不介绍了哈。

&emsp;&emsp;现在就开始新的写作记录旅程吧!

> 路漫漫其修远兮，吾将上下而求索(The road ahead will be long, I shall search)



### 参考

1. [shortcode-templates](https://gohugo.io/templates/shortcode-templates/) 
