---
title: Start Blog
date: 2016-03-27 15:32:01
tags: [hexo,博客,随笔,github]
---

## 前言
平时看博客，发现很多人 csdn，cnblog 的博客不更新了，都开始自己建个人博客，反正想着最近要沉淀下来，开始把一些东西写下来，不仅仅是保存到云笔记上，于是开始走起来。

## 开始
周末闲暇时间，尝试用 node js，hexo 搭建一个[ github博客](https://SeaZhang.github.io),

<!-- more -->

中间过程安装 hexo 过程中，有出现如下问题：

{ [Error: Cannot find module './build/Release/DTraceProviderBindings'] code: 'MODULE_NOT_FOUND' }    
{ [Error: Cannot find module './build/default/DTraceProviderBindings'] code: 'MODULE_NOT_FOUND' }
{ [Error: Cannot find module './build/Debug/DTraceProviderBindings'] code: 'MODULE_NOT_FOUND' }

google 一下尝试如下：

``` bash
$ npm install hexo --no-optional
```

中间又报出权限问题，命令前添加 sudo

解决了安装过程问题后，执行如下指令：

``` bash
$ hexo g
$ hexo s
```

浏览器打开 http://0.0.0.0:4000/


 哈哈哈，本机显示出来了

 ## 最后
本地显示 ok 后，开始上传到 github 上去。

hexo 安装 hexo-deployer-git

``` bash
npm install hexo-deployer-git --save
```

安装成功后进，修改 _config.yml 中 

deploy:
  type: git
  repo: <repository url>
  branch: [branch]
  message: [message]

配置完成后，
``` bash
hexo clean 
hexo g
hexo s
hexo d
```


## 后缀

建博过程中参考了如下博主的博客文章中的详细描述：
http://www.jianshu.com/p/fd878edb95e7
http://yidao620c.github.io/2016/03/06/hexo.html


