# Bucket Li Blog

一个使用 Astro 构建的简洁静态博客，部署在 GitHub Pages。

## 本地预览

需要 Node.js 22.12 或更高版本。

```bash
npm install
npm run dev
```

打开终端提示的本地地址即可预览。发布前可以运行：

```bash
npm run build
npm run preview
```

## 写文章

在 `src/data/posts/` 中新建 Markdown 文件，例如 `hello-world.md`：

```md
---
title: 文章标题
description: 一句话摘要
publishedAt: 2026-08-20
draft: false
---

这里是正文。
```

文件名会成为文章地址，例如上面的文件会生成 `/posts/hello-world/`。
将 `draft` 设为 `true` 时，文章不会出现在构建结果中。

## 发布

推送到 `master` 分支后，GitHub Actions 会自动构建并发布。首次使用时，需要在仓库的 **Settings → Pages → Build and deployment** 中将 Source 设为 **GitHub Actions**。
