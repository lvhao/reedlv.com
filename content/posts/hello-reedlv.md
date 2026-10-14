+++
title = "Hello, reedlv.com"
date = 2026-10-15T10:00:00+08:00
draft = false
tags = ["blog"]
categories = ["Notes"]
summary = "博客上线第一篇：记录站点从零到部署的过程。"
+++

这是 `reedlv.com` 的第一篇文章。

## 为什么选择 Hugo

- Markdown 写作，简单直接
- 构建速度非常快
- 部署到 Cloudflare Pages 后自动构建和发布

## 写作流程

```bash
hugo new posts/my-first-post.md
hugo server -D
```

发布时把文章的 `draft` 改为 `false`，推送到 GitHub 后 Cloudflare Pages 会自动部署。
