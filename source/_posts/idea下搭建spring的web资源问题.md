---
title: idea下搭建spring的web资源问题
date: 2018-09-06 22:36:37
tags: [idea, spring]
categories: java web
respository: git@github.com:Gfeather/feather.git
branch: master
---

*learning ,  perception design* 

#### 问题

在IDEA中配置好spring mvc之后Junit测试通过，但是部署在tomcat中404错误，而项目在Eclipse中可以正常运行。
{% asset_img 404.png %}

#### 分析
页面路径为`/WEB-INF/pages/*`因为Message显示路径为经过`Controller`处理、视图解析器解析好的路径，所以初步判断为资源访问不到，页面放到WEB-INF之外可以访问，但是是通过web容器直接访问，不经过spring mvc处理，所以没有考虑tomcat问题。
因为是Maven项目，不存在依赖问题，Maven打包的target中直接包含web源文件夹以及之下的WEB-INF，所以开始没有考虑到maven打包问题。

#### 进一步
中间部署查看配置发现项目的WEB资源目录发生了自动变更，以为是资源目录错误导致页面无法访问，但是更改之后重新部署依然无效。
{% asset_img webfact.png %}
IDEA中项目的Facets里可以配置各个模块的资源依赖，比如spring中会配置配置文件以及读取注解配置类
{% asset_img springfact.png %}
这个时候检查了一遍target目录，也就是Maven生成的War包，发现包下的WEB-INF目录下没有pages文件夹，所以问题根源确实是页面不存在。只是被资源文件夹误导，没有发现生产环境没有包含页面文件夹。

#### 解决
然后就开始找Web资源文件夹的配置，再次检查Facets中的Web资源文件夹配置，没有问题，检查IDEA项目配置中的War包同样没有。
开始怀疑Maven打包方式问题，BaiDu Maven打包时资源文件配置，添加标签后pom文件报错，放弃。
继续查看IDEA项目配置，检查artfacts部署文件内容怎么添加Web资源文件，然后发现可以添加JAVAEE Facet Resources,添加到创建的pages下，部署，生效。。。
{% asset_img artifacts.png %}
然后发现上一层目录，也就是WEB-INF的同级目录下自动添加了Web的资源文件，之前因为名字太长不知道用途，结果忽视。

#### 根源
Web应用开发中，目录、路径是很容易出错的细节，不同的IDE有不同的目录结构、路径读取规则。
