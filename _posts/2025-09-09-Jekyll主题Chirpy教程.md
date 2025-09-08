---
title: Jekyll主题Chirpy教程
author: yuzon
date: 2025-09-09
categories: [Github Page,Jekyll]
tags: [Github Page,Jekyll]
description: 
# toc: false 关闭提纲
# comments: false 关闭评论
---

## 目录结构   
    ├── _data/ 
    ├── _includes/
    ├── _layouts/
    ├── _plugins/
    ├── _posts/
    ├── _sass/
    ├── assets/
    │   ├── css/
    │   ├── images/
    │   └── js/
    ├── _config.yml
    ├── .gitignore
    ├── .jekyll-cache/
    ├── .jekyll-metadata
    ├── 404.html
    ├── about.md
    ├── Gemfile
    ├── Gemfile.lock
    ├── index.html
    └── README.md
## 目录介绍
* _data/: 存储站点数据文件，如导航菜单、社交链接等。  
* _includes/: 包含可重用的 HTML 片段，如页眉、页脚等。  
* _layouts/: 存储页面布局模板。  
* _plugins/: 存放自定义的 Jekyll 插件。  
* _posts/: 存放博客文章的 Markdown 文件。  
* _sass/: 包含 SASS 样式文件。  
* assets/: 存放静态资源，如 CSS、JavaScript 和图片。
* _config.yml: 项目的配置文件。  
* .gitignore: 指定 Git 忽略的文件和目录。  
* .jekyll-cache/: Jekyll 生成的缓存文件夹。  
* .jekyll-metadata: Jekyll 生成的元数据文件。
* 404.html: 404 错误页面。
* about.md: 关于页面。 
* Gemfile: 依赖管理文件。  
* Gemfile.lock: 依赖锁定文件。  
* index.html: 主页。  
* README.md: 项目说明文档。

## 启动文件介绍
Jekyll 主题 Chirpy 的启动文件主要是 index.html 和 _config.yml。  
### *index.html* 
index.html 是站点的主页文件，它使用 _layouts 目录下的布局模板来渲染页面内容。通常，index.html 会包含以下内容:  

    ---
    layout: home
    ---  
### *_config.yml*  
_config.yml 是 Jekyll 的主要配置文件，包含了站点的全局配置信息，如站点标题、描述、URL、作者信息等。以下是一些常见的配置项：

