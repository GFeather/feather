---
title: hexo搭建
date: 2018-08-29 17:23:31
tags: hexo
respository: git@github.com:Gfeather/feather.git
branch: master
---

{% asset_img  head.png %}
<!--more-->
*learning ,  perception design* 

具体搭建非常简单，安装环境、配置git、就可以部署
这里主要写使用next主题遇到的问题，部分问题还未解决，如果有解决方法可以评论告诉我。
*ps：评论就是我刚解决的问题(￣▽￣)~**

### 遇到的问题

1.集成gitment报凭证无效错误、variable failed
2.gitment显示在了标签和分类页
3.集成的leancloud无法显示阅读数，但是leancloud端可以看到访问数据

### 解决方案

目前只解决第一个问题，额。。。。
step1：
1.首先凭证无效问题，其实问题不大，步骤为网上大部分教程那样，我搞错的一点是配置文件中的`repo:`，我填了仓库地址，实际应该填仓库名就好
{% asset_img gitment.png %}
2.Variable failed 问题的原因也是网上很多博客说的那样，github的issue的标题长度限制在50字符，因为hexo在存储的时候使用的是`window.location.pathname`，但是这个字符串是经过编码的，如果你从console中打印出来会看到一串乱码(太久没玩CTF，忘记是什么编码了)，所以需要将`next\layout\_third-party\comments`下的`gitment.swig`文件中存储的`id: window.location.pathname`字段修改，使它的长度没那么长，解决方法有很多，目前发现的最优秀的方法是使用md5函数，因为我以为直接部署不用重新生成这个文件就会生效的原因，这个方案并不是最终解决方案，and。。。因为那个很zz的想法我错过了网上搜到的所有方案。
所以最终方案是将`window.location.pathname`使用`decodeURI()`方法将其解码，在使用正则表达式将日期到标题筛选出来
{% asset_img id.png %}

### 额
其他问题偶尔心情好会探索一下，如果你有好的解决方法，可以把解决方法分享一下。。。(〃'▽'〃)
