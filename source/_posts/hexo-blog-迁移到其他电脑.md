---
title: hexo blog 迁移到其他电脑
date: 2023-05-06 11:03:21
tags:
---
  新电脑配置环境，安环nodejs git hexo
  需要复制的文件和目录如下
  ![](/images/blog_mv_file.png)

执行以下命令
```
npm install
npm install hexo-deployer-git --save
npm install hexo-generator-feed --save
npm install hexo-generator-sitemap --save
```

验证
```
hexo g
hexo s
```
