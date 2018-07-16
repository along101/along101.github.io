---
title: hexo博客使用Travis CI自动发布
date: 2018-07-16 11:09:13
tags: [hexo,blog,travis,ci]
categories: 工具
---

# hexo 介绍
Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

搭建hexo博客可以参考：http://techblog.ppdai.com/2018/07/06/20180706/

**本文主要介绍如何使用Travis CI自动发布blog，免除了在本地搭建nodejs、安装hexo和手工发布blog的繁琐过程。**

<!--more-->

# 思路
hexo手工发布blog到github的原理是通过`hexo g`，将md文件生成静态的html文件，然后通过`hexo d`，将静态的html文件push到github仓库中。

这个过程都是在本地的nodejs环境中运行的，完全可以使用Travis来完成该过程。Travis是一个为GitHub服务的CI，可以把Travis看做一个远程服务器，它会监控你添加的GitHub库。如果有新的提交或Pull Request，就会触发一次在线编译，执行配置编译命令。

所以，可以将Hexo源码和blog静态页面放到github仓库的不同分支，在Hexo源码分支中添加Travis的配置，每当提交blog文档md后，Travis会自动生成静态页面并发push指定分支，完成blog的发布。

# hexo源码准备工作

1. 使用github搭建好个人blog
>参考http://techblog.ppdai.com/2018/07/06/20180706/

2. 创建分支存放hexo源码及blog源文件

在username.github.io中创建分支可以命名问hexo，将hexo源代码已经blog源文件提交到该分支。master为blog生成后的html静态页面分支。

# Travis设置
1.进入官网[https://www.travis-ci.org/](https://www.travis-ci.org/)，使用github账号登录。
2.添加仓库`username.github.io`

![](/images/travis/travis.png)

3. 在github添加Access Token，在travis中用此token push commit到github

 ![](/images/travis/github-token.png)

4. 在Travis设置页面将上一步生成的token添加到环境变量中，命名为myblog

![](/images/travis/travis-token.png)

5. 在`username.github.io`仓库hexo分支中添加'.travis.yml'配置文件

```yaml
language: node_js
node_js: stable

# S: Build Lifecycle
install:
  - npm install

#before_script:
  # - npm install -g gulp

script:
  - hexo g

after_script:
  - cd ./public
  - git init
  - git config user.name "along101"
  - git config user.email "yzl1414114@163.com"
  - git add .
  - git commit -m "Update docs"
  - git push --force --quiet "https://${myblog}@${GH_REF}" master:master

#  E: Build LifeCycle
branches:
  only:
    - hexo
env:
  global:
    - GH_REF: github.com/along101/along101.github.io.git
```
`${myblog}`为上一步添加的环境变量，需要配置自己的
- `user.name` github用户名
- `user.email` github注册邮箱
- `GH_REF` 仓库地址

6. 提交该配置到git仓库，Travis就会自动发布blog，以后写blog，只需要在hexo分支的`source/_posts`目录下提交blog的md文档，Travis就会自动发布静态页面到master上

![](/images/travis/travis-build.png)
