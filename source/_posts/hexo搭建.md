---
title: hexo搭建
date: 2018-06-01 09:24:54
tag: hexo
toc: true
description: 

---
## 开始搭建
### 1. 创建Git仓库
在git上创建存放blog的仓库，名称为 username.github.io
<br>
### 2. 安装homebrew
执行以下命令安装homebrew：
```
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
还可以查看homebrew [官方文档](https://brew.sh/)

<!-- more -->

安装Node.js:

```
$ brew link node
$ brew uninstall node
$ brew install node
```
安装hexo
```
$ sudo npm install hexo-cli -g
```
### 3. 创建博客
```
$ hexo init username.github.io
```
安装主题 (经典的next主题)
```
$ cd username.github.io
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```
博客和主题的配置都可以在对应文件下名为"_config.yml"的文件中修改.

新建博客文章可以在目录: /source/_posts下直接创建:
```
$ hexo new blogname
```
开启测试服务器:
```
$ hexo s
```
测试就可以访问: <https://localhost:4000>来查看本地博客内容了

将hexo部署到Git上:
```
$ npm install hexo-deployer-git --save
$ hexo clean && hexo g && hexo d
```

在文章中加入 `<!-- more --> `表示首页 **阅读全文** 的分界线

参考链接:
1. [mac环境下搭建hexo+github pages+next个人博客](https://juejin.im/post/5ac8db4d6fb9a028d5675c13)
2. [5分钟 搭建免费个人博客](https://www.jianshu.com/p/4eaddcbe4d12)
