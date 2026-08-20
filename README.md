# Bucket Li Blog

A minimal static blog built with Astro and deployed to GitHub Pages.

## Local development

Node.js 22.12 or later is required.

```bash
npm install
npm run dev
```

Open the local URL shown in the terminal. Before publishing, you can run:

```bash
npm run build
npm run preview
```

## Writing a post

Create a Markdown file in `src/data/posts/`, for example `hello-world.md`:

```md
---
title: Post title
description: A one-sentence summary
publishedAt: 2026-08-20
draft: false
---

Start writing here.
```

The filename becomes the post URL, so the example above is generated at `/posts/hello-world/`.
Set `draft` to `true` to exclude a post from the generated site.

## Deployment

Pushing to the `master` branch automatically builds and deploys the site. For the first deployment, set **Settings → Pages → Build and deployment → Source** to **GitHub Actions**.
