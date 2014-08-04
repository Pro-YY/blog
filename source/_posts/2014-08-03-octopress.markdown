---
layout: post
title: "使用octopress搭建github博客"
date: 2014-08-03 10:03:33 +0800
comments: true
categories: octopress
---

## 初始化安装

参考以下三篇文章

- [http://octopress.org/docs/deploying/github/](http://octopress.org/docs/deploying/github/)
- [http://chenzhiwei.net/2012/07/07/this-is-the-first-article/](http://chenzhiwei.net/2012/07/07/this-is-the-first-article/)
- [http://octopress.org/docs/setup/](http://octopress.org/docs/setup/)

### 本地安装octopress环境

    git clone git://github.com/imathis/octopress.git octopress
    cd octopress
    bundle install
    rake install


### 部署至github

核心思路: 部署仓库和源文档仓库分离。

1. 在github中新建两个repo:

- pro-yy.github.io: github的个人主页部署仓库，作为octopress \_deploy目录的upstream

- blog: 博客的源文件仓库，作为octopress source分支的upstream

2. 建立部署仓库

    rake generate
    rake deploy

此命令后，octopress完成对部署仓库(\_deploy目录下)的全部工作，包括生成文件、提交等等。
并且将源文件仓库初始化完成，设为source分支。但并不为该分支设提交。所以安装文档里有如下提示：

**Don't forget to commit the source for your blog.**

3. 为source分支设立upstream

    git remote remove origin
    git remote add origin git@github.com:Pro-YY/blog.git
 
4. CNAME 直接在source目录下添加即可

## 使用常用操作流程
 
### clone整个博客
 
    git clone git@github.com:Pro-YY/blog.git
    cd blog
    git clone git@github.com:Pro-YY/pro-yy.github.io.git _deploy


### 编辑流程

编辑文件

    rake new_post['post_name']
    rake new_page['page_name']

生成查看

    rake generate
    rake preview

提交源文件

    git push

提交部署文件

    rake deploy


### 添加个人介绍页面(About)

添加新页面并编辑

    rake new_page[about]

添加页面链接

```html source/_includes/custom/navigation.html
<ul class="main-navigation">
  <li><a href="{{ root_url }}/">Blog</a></li>
  <li><a href="{{ root_url }}/blog/archives">Archives</a></li>
  <li><a href="{{ root_url }}/about">About</a></li>
</ul>
```

### 添加disqus评论

参考[http://asaf.github.io/blog/2013/07/08/blogging-with-octopress-add-comments/](http://asaf.github.io/blog/2013/07/08/blogging-with-octopress-add-comments/)

### 更新octopress

    git remote add octopress https://github.com/imathis/octopress
    git fetch octopress
    git rebase octopress/master
