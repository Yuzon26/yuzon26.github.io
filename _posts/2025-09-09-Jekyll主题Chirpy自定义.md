---
title: Jekyll主题Chirpy自定义
author: yuzon
date: 2025-09-09
categories: [Github Page,Jekyll]
tags: [Github Page,Jekyll]
---
## 自定义网站图标  
网站图标位于目录`assets/img/favicons/`

准备一张大小为 512x512 或更大的方形图像（PNG、JPG 或 SVG），然后转到在线工具[**Real Favicon Generator**](https://realfavicongenerator.net/) 生成`.ico`文件。下载生成的包，解压并从解压的文件中删除以下两个：
- `browserconfig.xml`
- `site.webmanifest`  

然后复制剩余的图像文件（和）以覆盖 Jekyll 站点目录中的原始文件。
