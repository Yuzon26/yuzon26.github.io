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

- lang - 语言 变量取值详见[ISO语言代码](http://www.lingoes.net/en/translator/langcode.htm)（如英语的lang取值为en，中文的lang取值为zh-CN）
- timezone - 时区
- title - 网站标题
- tagline - 网站副标题
- description - 网站描述
- url - 部署博客的地址，如`https://USERNAME.github.io`
- avatar - 作者头像，如`https://chirpy-img.netlify.app/commons/avatar.jpg`



## `./_data/`可选配置
在子目录`./_data/`下还存放有一些可选填的配置文件，主要用于设置网页的外观，可根据需求修改。以下是对这些配置文件的简要说明：
### layout text 语言配置
语言配置存放在`_data/locate/`，其中英语的配置文件为`_data/locate/en.yml`，中文的配置文件为`_data/locate/zh-CN.yml`。如下面这段配置定义了英文语言下侧边菜单栏的选项名称。

```yaml
# ----- Commons label -----
layout:
  post: 文章
  category: 分类
  tag: 标签
# The tabs of sidebar
tabs:
  # format: <filename_without_extension>: <value>
  home: 首页
  categories: 分类
  tags: 标签
  archives: 归档
  about: 关于
```
### 作者信息
存放在`_data/authors.yml`，可填写多个作者。例：

```yaml
# {author_id}:
#   name: {full name}
#   twitter: {twitter_of_author}
#   url: {homepage_of_author}
cotes:
  name: Cotes Chung
  twitter: cotes2020
  url: https://github.com/cotes2020/
```
### 侧边栏社交账号信息设置
存放在`_data/contact.yml`，有gitHub、twitter、email、rss（用于订阅网站内容更新，它允许用户无需访问网站，就能获取最新的内容）。例：
```yaml
- type: github
  icon: "fab fa-github"

- type: twitter
  icon: "fa-brands fa-x-twitter"

- type: email
  icon: "fas fa-envelope"
  noblank: true # open link in current tab

- type: rss
  icon: "fas fa-rss"
  noblank: true
```
### 分享信息配置
存放在`_data/share.yml`，分享方式有Twitter、Facebook、Telegram等。
```yaml
platforms:
  - type: Twitter
    icon: "fa-brands fa-square-x-twitter"
    link: "https://twitter.com/intent/tweet?text=TITLE&url=URL"

  - type: Facebook
    icon: "fab fa-facebook-square"
    link: "https://www.facebook.com/sharer/sharer.php?title=TITLE&u=URL"

  - type: Telegram
    icon: "fab fa-telegram"
    link: "https://t.me/share/url?url=URL&text=TITLE"
```

## `./_sass/`可选配置
### 字体配置
样式文件存放在`_sass/abstracts/_variables.scss`每种语言对应一套默认字体，如英语默认的标题字体为Lato，段落字体为Source Sans Pro，中文字体可改为PingFang SC，更多参考这篇[说明](https://github.com/cotes2020/jekyll-theme-chirpy/pull/986)。
```scss
/* fonts */
$font-family-base: 'Source Sans Pro', 'Microsoft Yahei', 'PingFang SC', sans-serif !default;
$font-family-heading: Lato, 'Microsoft Yahei', 'PingFang SC', sans-serif !default;
```

## 文档语法
### Front Matter 
```
    ---
    title: 文章标题
    date: 2025-09-09 +0800
    categories: [社区管理, 帮助] #上级文文档，下级文档，两级
    tags: [社区管理] 标签个数不限
    description: . # 描述
    author: # 作者信息
      name: Full Name
      link: https://example.com
    toc: false # 关闭目录
    comments: false # 关闭评论
    math: true # 加载数学功能
    mermaid: true # 启用Mermaid
    pin: true # 置顶帖子
    media_subpath : /assets/img/in-post/2024/2024-08-08/
    image:
        path: idempiere11_img_back.jpg
    ---
```

## 参考网址
- [iDempiere社区](https://ittousei.github.io/)
- [Chirpy官网语法讲解](https://chirpy.cotes.page/posts/write-a-new-post/#table-of-contents)
- [去以六月息](https://ittousei.github.io/)