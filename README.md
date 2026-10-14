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
git commit -m "post: my note"
git push
```
