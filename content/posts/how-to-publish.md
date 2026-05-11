---
title: "如何发布博客文章"
date: 2026-05-11
tags: ["教程"]
categories: ["博客"]
summary: "介绍向本博客提交文章的几种方式。"
ShowToc: true
---

本博客基于 Hugo + PaperMod 搭建，托管在 GitHub Pages 上。以下是发布文章的几种方式。

## 方式一：本地写文章 + Git 推送

适合认真写长文的场景。

1. 在 `content/posts/` 目录下创建 `.md` 文件
2. 写入以下内容：

```markdown
---
title: "文章标题"
date: 2026-05-11
tags: ["标签"]
categories: ["分类"]
summary: "文章摘要"
ShowToc: true
---

正文内容，支持标准 Markdown 语法。
```

3. 提交并推送：

```bash
git add content/posts/xxx.md
git commit -m "Add: 文章标题"
git push
```

推送后 GitHub Actions 会自动构建部署，1-2 分钟后即可在网站上看到。

## 方式二：GitHub 网页直接创建

最简单的方式，随时随地可用。

1. 打开仓库页面：https://github.com/WayneGreat/Blog
2. 进入 `content/posts/` 目录
3. 点击 **Add file → Create new file**
4. 文件名填写 `xxx.md`（英文命名）
5. 写入文章内容（需要包含 frontmatter）
6. 点击 **Commit changes**

无需本地环境，浏览器操作即可。

## 方式三：VS Code 编辑 + 提交

适合追求写作体验的场景。

1. 用 VS Code 打开项目目录
2. 安装 Markdown 预览插件（如 Markdown Preview Enhanced）
3. 在 `content/posts/` 下新建 `.md` 文件
4. 边写边预览，满意后用 VS Code 内置 Git 提交推送

## 方式四：Hugo 命令生成模板

适合本地已安装 Hugo 的场景。

```bash
hugo new posts/my-post.md
```

会自动在 `content/posts/` 下生成带 frontmatter 的模板文件，编辑后推送即可。

## Frontmatter 参数说明

每篇文章开头必须包含 frontmatter，常用参数如下：

| 参数 | 说明 | 示例 |
|------|------|------|
| `title` | 文章标题 | `"我的文章"` |
| `date` | 发布日期 | `2026-05-11` |
| `tags` | 标签 | `["技术", "教程"]` |
| `categories` | 分类 | `["编程"]` |
| `summary` | 摘要 | `"文章简介"` |
| `ShowToc` | 显示目录 | `true` / `false` |
| `math` | 启用数学公式 | `true` / `false` |
| `draft` | 草稿（不发布） | `true` / `false` |

## 方式对比

| 方式 | 优点 | 缺点 |
|------|------|------|
| 本地 Git 推送 | 完全控制，适合长文 | 需要本地环境 |
| GitHub 网页 | 随时随地，无需环境 | 编辑体验一般 |
| VS Code | 写作体验好，实时预览 | 需要安装软件 |
| Hugo 命令 | 自动生成模板 | 需要本地安装 Hugo |

推荐日常短文用 **方式二**（GitHub 网页），认真写长文用 **方式一** 或 **方式三**。
