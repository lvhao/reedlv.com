# reedlv.com

个人博客，基于 Hugo + PaperMod，部署到 Cloudflare Pages。

## 本地开发

```bash
hugo server -D
```

## 发布文章

```bash
hugo new posts/my-note.md
# 编辑内容后，将 draft 改为 false
git add .
git commit -m "新增文章：my note"
git push
```

push 后 GitHub Actions 自动构建并部署到 Cloudflare Pages（依赖仓库 Secrets：`CLOUDFLARE_API_TOKEN`、`CLOUDFLARE_ACCOUNT_ID`）。

本地兜底部署：

```bash
hugo --minify && npx wrangler pages deploy public --project-name=reedlv --commit-dirty=true
```

