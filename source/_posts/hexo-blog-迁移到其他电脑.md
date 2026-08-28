---
title: hexo blog 迁移到其他电脑
date: 2023-05-06 11:03:21
categories: Hexo
tags:
	- hexo
	- 迁移
---

换电脑后把 Hexo 博客再跑起来，不必重装主题、不必重写文章。源码在 Git 里时，克隆下来安装依赖即可；如果只是拷贝目录，带上配置、文章和主题，去掉生成物。

当前站点发布在 GitHub Pages：<https://seamonw.github.io/blog/>。源码仓库是 [seamonw/blog](https://github.com/seamonw/blog)，推到 `main` 后由 GitHub Actions 构建。

## 新电脑要装的环境

- [Node.js](https://nodejs.org/)（LTS 即可，本仓库按 Node 20+ 验证过）
- [Git](https://git-scm.com/)
- npm（随 Node 一起安装）

装完后确认：

```bash
node -v
npm -v
git --version
```

Hexo 不必全局安装。仓库里已经有 `hexo` 依赖，后面用 `npx hexo` 或 `npm run` 即可。

## 推荐做法：直接克隆仓库

这是现在最省事的方式，主题、文章、GitHub Actions 工作流都会一起过来：

```bash
git clone https://github.com/seamonw/blog.git
cd blog
npm ci
```

没有 `package-lock.json` 时改用 `npm install`。

## 只拷贝目录时带哪些文件

不走 Git、只从旧电脑拷文件夹时，带上这些即可：

![迁移时需要复制的文件](/images/blog_mv_file.png)

对应到当前仓库，需要保留：

| 路径 | 作用 |
| --- | --- |
| `_config.yml` | 站点配置，含 `url` / `root` / 主题名 |
| `package.json`、`package-lock.json` | 依赖版本 |
| `source/` | 文章、图片、分类和标签页 |
| `themes/next/` | NexT 主题（含本地改过的 `_config.yml`） |
| `scaffolds/` | `hexo new` 用的模板 |
| `.github/workflows/` | GitHub Pages 自动构建 |
| `.gitignore` | 避免把 `node_modules`、`public` 提交进去 |

不要拷、也不要提交：

- `node_modules/`：在新电脑重新安装
- `public/`：用 `hexo generate` 再生成
- `.deploy_git/`：旧的 git 部署缓存
- `db.json`：Hexo 本地缓存，可自动重建

拷完后在项目根目录执行：

```bash
npm install
```

下面几个生成器和部署插件已经写在 `package.json` 里，`npm install` 会一并装上，不必再单独敲：

```bash
npm install hexo-deployer-git --save
npm install hexo-generator-feed --save
npm install hexo-generator-sitemap --save
```

只有在你拿的是一份很老的目录、`package.json` 里还没有这些包时，才需要补这几行。

## 本地验证

生成静态文件并打开本地预览：

```bash
npx hexo clean
npx hexo generate
npx hexo server
```

或用 npm 脚本：

```bash
npm run clean
npm run build
npm run server
```

浏览器打开 <http://localhost:4000/blog/>。因为站点 `root` 是 `/blog/`，本地也走这个子路径，不要只访问 `http://localhost:4000/`。

常见检查：

1. 首页能列出文章
2. 点进一篇正文，代码高亮和图片正常
3. `/blog/archives/`、`/blog/categories/`、`/blog/tags/` 能打开

## 写文章和发布

```bash
npx hexo new "文章标题"
```

文件会出现在 `source/_posts/`。改完后先 `hexo server` 看效果，再提交推送：

```bash
git add source/_posts
git commit -m "Add a new post."
git push origin main
```

GitHub Actions 会执行 `npm ci` 和 `hexo generate`，把 `public/` 发布到 GitHub Pages。几分钟后到 <https://seamonw.github.io/blog/> 确认。

不要把 GitHub Token 写进 `_config.yml`。以前用 `hexo-deployer-git` 一键部署时容易把 Token 嵌进仓库地址，有泄露风险。现在用 Actions，仓库里只保留源码即可。

## 迁移后核对清单

- `_config.yml` 里 `url` 为 `https://seamonw.github.io/blog`，`root` 为 `/blog/`
- `theme: next`，且 `themes/next` 目录存在
- `source/_posts/` 里文章都在
- `git remote -v` 指向 `https://github.com/seamonw/blog.git`
- 推送后 Actions 的 Pages 工作流是绿色

环境配好、目录拷全、本地 `hexo s` 能看，迁移就完成了。之后换电脑重复“克隆 + `npm ci` + `hexo s`”即可。
