---
title: 开启我的博客
tags: [Hexo, GitHub, Blog]
categories: 
- Blog
category_map:
 博客基础: Blog
toc: true
---
### 基础环境配置

 1. node.js 安装
 2. git 环境安装
 3. hexo 安装 npm install -g hexo-cli 权限不够请使用 sudo npm install -g hexo-cli

### GitHub 创建博客关联创库

 1. 在gitHub上创建一个创库，起名your_github_name.github.io

### 搭建博客

 1. 创建属于你的博客文件夹xxxBlog 并初始化
 
    - cd xxxBlog
    - hexo init

 2. 导入依赖包

    - npm install

 3. 在上述步骤生成的_config.yml中定制配置
    - itle: Choose a title
subtitle: Any subtitle you like
description: Anything you like
author: Your name
language: zh-CN
timezone: Asia/Shanghai
deploy:
type: git
repo: https://github.com/your_github_name/your_github_name.github.com.git
branch: master

### 博客部署

 1. 开启本地验证
    - hexo server

 2. 安装部署器
    - npm install hexo-deployer-git --save

 3. 博客部署到GitHub上
    - hexo deploy

 4. 浏览器your_github_name.github.io访问部署以后的博客

### 博客主题切换

 1. 挑选自己喜欢的主题，Google上随便一搜hexo的主题多的是，我自己使用的是Maupassant的主题，屠夫的优化版。
    - git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant

 2. 安装主题和渲染器
    - npm install hexo-renderer-jade --save
    - npm install hexo-renderer-sass --save
    
 3. 重新编辑xxxBlog下的_config.yml文件
    - 将theme的值改为maupassant

 4. hexo三部曲
    - hexo clean
    - hexo g
    - hexo deploy

 5. 浏览器your_github_name.github.io访问部署以后的新主题的博客

### 404页面配置

 1. 在source下创建一个404.html的文件
 2. 使用腾讯公益404代码，直接在html中写如下代码
    ```<script type="text/javascript" src="http://www.qq.com/404/search_children.js" charset="utf-8"></script>```
    
### 适配多电脑发布博客

 1. 在github的your_github_name.github.io上创建一个分支，名hexo
 2. 将hexo设置为默认分支
 3. git clone your_github_name.github.io 到本地，然后将xxxBlog里面的文件都拷贝到your_github_name.github.io中，将theme中的.git文件删除，git add/commit 这些变更。
 4. 以后只要git pull这个分支，然后npm install就可以在其他的电脑上操作你的博客了。

### Hexo 的基本操作
 
 

 1. 创建新博客
     - hexo new "create Blog"

 2. 创建单独页面
 
    - hexo new page xxx

 3. 本地构建服务
 
    - hexo server
 4. 本地html缓存清理
 
    - hexo clean
 5. html 生成
 
    - hexo generate
 6. html 部署
 
    - hexo deploy



