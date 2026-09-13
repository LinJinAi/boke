# boke

基于 Hugo + GitHub Pages 的静态博客 (◕‿◕)

## 结构

```
.
├── content/posts/        # 文章目录，新增 Markdown 文章放这里
├── layouts/              # 主题模板（内置精简主题）
├── config/_default/
│   └── config.yaml       # 站点配置
└── .github/workflows/
    └── hugo.yml          # 部署到 GitHub Pages
```

## 文章 frontmatter

每篇文章以 YAML frontmatter 开头，包含四个字段：

```markdown
---
title: 文章标题
date: 2026-09-13
tags: [hugo, blog]
author: LinJin
---

正文 Markdown 内容……
```

## 发布流程

1. 在 `content/posts/` 下新增 Markdown 文章
2. 提交 PR（推荐用 `.github/pull_request_template.md`）
3. PR 合并到 `main` 分支
4. GitHub Actions 自动构建并部署到 GitHub Pages

仓库需启用 Pages：Settings → Pages → Build and deployment → Source 选择 **GitHub Actions**。

部署地址（构建完成后）：
`https://LinJinAi.github.io/boke/`
