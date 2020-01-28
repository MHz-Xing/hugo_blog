---
date: 2019-10-21T17:18:56+08:00
publishdate: "2019-10-23"
lastmod: "2019-10-29"
draft: false
title: "Win+Hugo+GitHub搭建静态Blog[MD格式]"
tags: ["前端", "hugo", "GitHub"]
series: ["网站搭建"]
categories: ["技术博客"]
toc: true
---

**作为一个前端小白，记录搭建个人主站的流程，以备后查，也希望给有相同需求的小伙伴一些启发。**

<!--more-->
# Tools
## 1、Hugo
hugo官网 [https://gohugo.io/](https://gohugo.io/)

发布版本 [https://github.com/gohugoio/hugo/releases](https://github.com/gohugoio/hugo/releases)

本文使用0.58.3版本

![版本信息](/images/blog/hugo_version.png)

## 2、Git

# 安装
## 1、安装Git
绝大部分程序猿应该都知道，这里不再介绍
## 2、安装hugo
下载解压之后，将hugo.exe配置进环境变量。
在自己的工作目录下运行，blog为实际的目录
## 3、创建站点文件

```
hugo new site blog
```
## 4、安装主题

下载主题
地址：[https://themes.gohugo.io/](https://themes.gohugo.io/)

下载好对应的themes文件，放置到对应的Blog目录下，目录结构如下：

![目录树](/images/blog/file_def.png)

## 5、新建文章
  
 hugo new blog/2019-10/blog1.md -k blog ：新建一篇博客
 
 hugo new picture/picture1.md -k picture ：新建一个影集

## 6、本地调试

```
hugo server --buildDrafts
```

## 7、编译正式版本

```
hugo  -D
```

## 8、部署到GitHub

github 个人主站上首先创建个人的github page 仓库

注意：创建的名称必须为如下格式：

==你的github用户名.github.io==

eg：
```
zhangsan.github.io
```


进入本地public目录，分别执行以下操作

git init

git add .

git commit -m "git push v0.1 test"

git remote add origin git@github.com:XX/xx.github.io.git

git push -u origin master

如果上述操作不成功，可以先将remote端在服务器上新建一个readme文件，之后使用服务器自带的pull方法下载到本地，之后再执行git add .等指令

## 9、访问个人主页

xx.github.io

本站地址：[MHz Xing](https://MHz-Xing.github.io)

# 写在最后

## 常用命令
- hugo new site SITE_NAME 生成静态博客项目
- hugo server 启动本地服务器，加上 -D 可以渲染 draft
- hugo new post/new-content.md 在 post 下新建一篇文章
- hugo new blog/2019-10/blog1.md -k blog  在 blog/2019-10/下新建一篇名称为blog1的文章
- hugo new picture/picture1.md -k picture 在 picture//下新建一篇名称为picture1的相册集
- hugo 生成站点静态文件
- hugo list drafts/expired/future 列出特点的文件


