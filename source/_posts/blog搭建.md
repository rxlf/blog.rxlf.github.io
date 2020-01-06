---
title: blog搭建
date: 2019-09-08 11:53:40
tags: 
- hexo
---

blog已经搭建好有一段时间了，但是经历了找工作，出去玩，新入职，搬家一系列事情，加上比较懒，一直没有写文章，所以这里总结一下blog搭建过程，还有一些遇到的坑

### 环境搭建

node.js (Should be at least nodejs 6.9)
git

配置好node和git后安装Hexo执行下面命令
> npm install -g hexo-cli

### 建站
配置好环境过后，就可以开始搭建我们的blog了
> hexo init myBlog
cd myBlog
npm install 

`myBlog`是创建文件夹的名称，可以根据自己的喜好来，建议以blog.xxxx命名，比如我的[blog.rxlf.github.io](https://github.com/rxlf/blog.rxlf.github.io)  
执行完上面的命令后，就可以开始写我们的blog了。

<!-- more -->

在写Blog之前，先简单的介绍一下初始化工程后的目录结构，以便知道各模块的作用：
```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

_config.yml 网站的配置信息，包括网站的名称语言环境，和后续发布等相关的内容。
package.json 应用程序信息
scaffolds 模板文件，Hexo 会根据 scaffold 来建立文件。
source 资源文件夹是存放用户资源的地方。
themes 主题文件夹。Hexo 会根据主题来生成静态页面。

### 开始写作
> hexo new post blog搭建

这样就建好了一篇“blog搭建”为title的文章，接下来就是markdown语法补充我们的内容就可以了。

### 本地生成查看效果
在写好我们的blog过后，如何查看文章效果
首先生成静态文件，命令如下
> hexo generate

查看效果
> hexo server

该命令会在本地起一个服务，默认连接为http://localhost:4000 ,复制到浏览器，打开就能看到文章效果。

### 博客配置
打开发现blog的站点名，title, subtitle 都不是我们自己的名字。这时候只需要在`config.yml`相应修改就可以了。

#### themes
默认主题很丑。。所以如果有需求，我们可以更换主题。我自己使用的是next，其他主题可以去官网https://hexo.io/themes/ 查看。这里介绍一下next安装：  
在终端下:
```
cd myBlog
git clone https://github.com/iissnan/hexo-theme-next themes/next
```
然后在`config.yml`,找到theme修改为next就可以了，更多详细的配置参考[next](http://theme-next.iissnan.com)

### 部署到github
前面已经讲了如何建站以及如何生成我们的blog文章以及一些基本配置，但是现在只能通过本地访问，我们写blog的目的还是让别人能访问到我们的blog。
那么如何让别人能访问我们的blog，这里选择托管到github上面，具体操作如下:  

首先创建github仓库，命名`xxx.github.io`, xxx为你的blog名字，比如前面的`myBlog`, 然后在仓库setting中开启Github Page  
这里有一点需要注意，Github Page 默认分支只能是master，当我们发布时，master分支是构建好的Public文件件中的内容，没有我们的源码，所以建议创建一个sourse分支，并且修改为默认分支，这样我们可以把源文件放到sourse分支，发布内容放到master，换了电脑和环境过后依然可以通过clone代码仓库找到我们的源文件，否则源文件就丢掉了。  

将仓库上传后，修改`config.yml`
```
url: https://github.com/rxlf/blog.rxlf.github.io.git
root: /blog.rxlf.github.io/
permalink: :year/:month/:day/:title/
permalink_defaults:
```
注意root一定要添加，否则css会加载失败。  
并且在`config.yml`添加, repo 就是你的github仓库地址，branch为发布分支
```
deploy:
  type: git
  repo: https://github.com/rxlf/blog.rxlf.github.io.git
  branch: master
```

当配置好了过后，就可以进行发布了，发布前需要安装`hexo-deployer-git`，否则会提示git仓库找不到，安装命令如下  
`npm install hexo-deployer-git --save`
完成安装之后，再次执行`hexo g`和`hexo d`命令(分别是hexo generate 和 hexo deploy的简写)。就可以发布到我们的github仓库了。查看Github Page上提示的url，访问这个链接，就能看到我们的blog啦。  

具体的命令请参考[hexo](https://hexo.io/zh-cn/docs/index.html)

### 访问自己的域名
首先申请自己的域名，阿里云或者腾讯云上申请，我自己是在腾讯云上申请的，所以这里以腾讯云为例  

配置解析  

添加二级域名  

在github page中修改域名

在`config.yml`修改
```
url: https://blog.rxlfchen.com
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:
```
### 如何能在google中搜到自己的blog

安装sitmap站点地图生成插件
`npm install hexo-generator-sitemap --save`
在配置文件中增加
```
# sitemap
sitemap:
  path: sitemap.xml
```
然后重新发布我们的blog，就可以了，其他还有解析配置方式等，可以参考google search里提示的方法

