# blog

seamonw 的个人博客源码。基于 [Hexo](https://hexo.io/) + [Butterfly](https://butterfly.js.org/) 主题，推送到 `main` 后由 GitHub Actions 自动构建并发布到 GitHub Pages。

站点地址：<https://seamonw.github.io/blog/>

[![Pages](https://github.com/seamonw/blog/actions/workflows/pages.yml/badge.svg)](https://github.com/seamonw/blog/actions/workflows/pages.yml)

## 技术栈

| 项目 | 说明 |
| --- | --- |
| 静态站点框架 | Hexo 6.3 |
| 主题 | Butterfly 5.7（npm 安装，配置覆盖在 `_config.butterfly.yml`） |
| 本地搜索 | `hexo-generator-searchdb`，生成 `search.xml` |
| 字数与阅读时长 | `hexo-wordcount` |
| RSS / 站点地图 | `hexo-generator-feed`、`hexo-generator-sitemap` |
| 部署 | GitHub Actions → GitHub Pages |
| 评论 | [Giscus](https://giscus.app/zh-CN)，评论存在仓库的 Discussions |

## 本地开发

需要 Node.js 20 及以上和 Git。Hexo 不用全局安装，仓库依赖里已包含。

```bash
git clone https://github.com/seamonw/blog.git
cd blog
npm ci
npm run server
```

打开 <http://localhost:4000/blog/>。

站点部署在子路径下（`root: /blog/`），本地预览也走这个路径，直接访问 `http://localhost:4000/` 会 404。

常用命令：

```bash
npm run clean    # 清理 public 和缓存
npm run build    # 生成静态文件到 public/
npm run server   # 本地预览，带热更新
```

## 写文章

```bash
npx hexo new "文章标题"
```

新文件出现在 `source/_posts/`，front-matter 至少填 `title` 和 `date`：

```yaml
---
title: 文章标题
date: 2026-08-28 19:10:00
categories: RPC
tags:
	- rpc
	- golang
---
```

正文里的站内资源写成站点内路径即可，Hexo 会按 `root` 自动补前缀：

```markdown
![示意图](/images/blog_mv_file.png)
[归档](/archives/)
```

图片放在 `source/images/`。

写完本地 `npm run server` 确认无误，再提交推送：

```bash
git add source/_posts
git commit -m "Add a new post."
git push origin main
```

## 目录结构

```text
.
├── _config.yml              # 站点配置：URL、永久链接、插件（feed / sitemap / search）
├── _config.butterfly.yml    # 主题配置，覆盖 node_modules 里 Butterfly 的默认值
├── package.json             # 依赖与 npm 脚本
├── scaffolds/               # hexo new 使用的文章模板
├── source/
│   ├── _posts/              # 文章
│   ├── categories/index.md  # 分类页
│   ├── tags/index.md        # 标签页
│   └── images/              # 图片资源
├── themes/next/             # 旧的 NexT 主题，保留作为回退方案
└── .github/workflows/       # GitHub Actions 构建与发布
```

`node_modules/`、`public/`、`db.json` 都不入库，由构建生成。

## 配置说明

分两个文件，不要去改 `node_modules` 里的主题文件：

- `_config.yml`：站点级配置。站点标题、作者、URL 与 `root`、永久链接格式，以及 `feed` / `sitemap` / `search` / `symbols_count_time` 这些插件的参数
- `_config.butterfly.yml`：主题级配置。导航菜单、顶部大图与文章封面、配色、深色模式、侧栏卡片、搜索、字数统计与评论

评论系统是 Giscus，对应仓库 `seamonw/blog` 的 Discussions（分类 Announcements）。读者用 GitHub 账号登录后即可评论。第一次启用时需要把 [Giscus 应用](https://github.com/apps/giscus) 安装到这个仓库，否则评论框会提示仓库未授权。

Hexo 会把 `_config.butterfly.yml` 深合并到主题自带的 `_config.yml` 之上，所以这个文件只需要写要覆盖的项。

想切回 NexT 主题，把 `_config.yml` 里的 `theme` 改回 `next` 即可。

## 部署

`.github/workflows/pages.yml` 在推送到 `main` 时触发：`npm ci` → `npm run build` → 把 `public/` 作为 Pages artifact 上传并发布。仓库的 Pages 发布源已设为 GitHub Actions。

也支持在 Actions 页面手动触发（`workflow_dispatch`）。

不要在 `_config.yml` 里写 GitHub Token。仓库只保留源码，发布由 Actions 用自带的权限完成。
