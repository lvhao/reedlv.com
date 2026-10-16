---
name: reedlv-release
description: 把 reedlv.com 仓库（~/Desktop/reedlv.com）的改动发上线——构建校验、按仓库风格提交、推送、部署到 Cloudflare Pages、线上复验。当用户说「提交」「推送」「部署」「发版」「上线」「ship it」「deploy」时使用。即使用户只说了其中一步，也按整条链走一遍再汇报。
---

# reedlv.com 发布流程

以下步骤绑定执行：校验 → 提交 → 推送 → 部署 → 复验，跳过哪一步都会咬回来。

## 1. 构建校验

`hugo --minify` 零报错才算过。

## 2. 提交

按仓库既有风格：新增文章用「新增文章：<标题>」；其他改动用中文一句话概括（如「业务复盘：润色开头，现状题记加前缀」）。不混入无关文件。

## 3. 推送

`git push`（main 分支）。

## 4. 部署

按优先级：

1. **GitHub Actions（首选，已配置）**：push 触发 `deploy.yml` 自动构建并部署。用 `gh run list --workflow=deploy.yml --limit 1` 确认最新 run 结束且为 success；未结束时等待或 `gh run watch`。
2. **本地 wrangler（Actions 失败/未生效时的兜底）**：

   ```bash
   hugo --minify && npx wrangler pages deploy public --project-name=reedlv --commit-dirty=true
   ```

## 5. 线上复验

curl 本次发布对应的 URL（新文章 = `https://reedlv.com/posts/<slug>/`）必须 200，且 `https://reedlv.com/index.xml` 包含新内容。失败则如实回报，不要假装成功。

## 注意

- Pages 项目 `reedlv` 是 Direct Upload 模式，Git Provider 为 No，push 本身不触发部署——靠 Actions（或本地 wrangler）。
- wrangler 已在本机登录；若提示缺 OAuth scope，`npx wrangler login` 刷新。
