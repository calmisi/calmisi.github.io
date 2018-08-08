---
title: re-init hexo-blog project from github
date: 2018-08-08 23:40:19
tags: 
- Git
- Hexo
categories: Hexo&NEXT
---

当换了新电脑后，如何将以前基于Hexo搭建的blog重新搭建呢。

之前的Hexo放在github上，并使用github.io展示。
其中项目名字为calmisi.github.io,包含两个branch: hexo和master
hexo为默认的branch，其保存hexo项目的文件，
master为展示branch，which holds the html codes that generates by `hexo generate`

- 安装Hexo环境
1. 首先安装Node.js，去官方下载安装，
2. 安装Hexo, `npm install -g hexo-cli`
3. 安装hexo-server, `npm install hexo-server --save`
4. 安装hexo-deploy-git, `npm install hexo-deploy-git --save`

- 克隆github上的blog项目
```
git clone git@github.com:calmisi/calmisi.github.io.git
```

- 初始化子模块，NEXT主题以子模块的形式放在另一个仓库
```
cd calmisi.github.io
git submodule init
git submodule update
```

- 初始化Hexo项目
```
npm install
```