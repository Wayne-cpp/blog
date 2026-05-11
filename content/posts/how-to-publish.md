---
title: "如何发布文章"
date: 2026-05-11
tags: ["Hugo", "教程"]
categories: ["博客"]
summary: "介绍在本博客中创建、预览和发布文章的完整流程。"
ShowToc: true
---

## 创建文章

在 `content/posts/` 目录下新建一个 `.md` 文件，添加 frontmatter 头部信息：

```yaml
---
title: "文章标题"
date: 2026-05-11
tags: ["标签"]
categories: ["分类"]
summary: "摘要"
ShowToc: true
---
```

然后正常书写 Markdown 正文即可。

### 可选参数

| 参数 | 说明 |
|---|---|
| `math: true` | 启用 KaTeX 数学公式渲染 |
| `draft: true` | 标记为草稿，不会被发布 |
| `author` | 作者名称 |
| `description` | 页面描述（SEO） |

Markdown 中可以直接嵌入原生 HTML（已开启 `unsafe: true`）。

## 本地预览

安装 Hugo extended 后，启动本地开发服务器：

```bash
hugo server -D
```

`-D` 参数会包含草稿文章。浏览器访问 `http://localhost:1313/blog/` 即可预览。

> 如果本地没有安装 Hugo，可以跳过此步，直接推送让 CI 构建。

## 提交发布

```bash
git add content/posts/你的文章.md
git commit -m "Add: 文章标题"
git push origin main
```

推送到 `main` 分支后，GitHub Actions 会自动构建并部署到 `gh-pages` 分支，文章随即上线。

## 删除文章

```bash
git rm content/posts/要删除的文章.md
git commit -m "Remove: 文章标题"
git push origin main
```

## 注意事项

- 提交信息使用中文前缀：`Add:`、`Fix:`、`Update:`、`Remove:`
- 仅修改 `images/`、`LICENSE`、`README.md` 不会触发部署
- 如果主题目录为空，执行 `git submodule update --init --recursive`
