# winlee.github.io

个人技术博客，基于 Hexo + [hexo-theme-typo](https://github.com/rankangkang/hexo-theme-typo)，发布在 <https://winlee.github.io>。

## 分支说明

| 分支 | 内容 |
|---|---|
| `source` | Hexo 项目源码（配置、文章、主题、脚手架） |
| `master` | `hexo deploy` 生成的静态站点，GitHub Pages 从这里发布 |

不要手改 `master`，也不要提交 `public/`、`db.json`、`.deploy_git/`——它们都是生成物，已在 `.gitignore` 中。

## 目录结构

```text
_config.yml          # Hexo 主配置（站点信息、permalink、deploy）
_config.typo.yml     # 主题配置覆盖（标题、菜单、分类下拉）
scaffolds/           # 新建文章/页面的模板
source/
  _posts/            # 文章 markdown
  categories/        # 各分类的列表页
  about/             # 关于页
  .nojekyll          # 跳过 Jekyll（随产物复制到 public/）
themes/typo/         # 内置主题（含自定义的分类下拉和 category 布局）
public/              # 生成产物（不提交）
```

## 本地开发

```bash
npm install
npx hexo server        # http://localhost:4000
```

## 写文章

```bash
npx hexo new "文章标题"
```

或直接创建 `source/_posts/<date>-<slug>.md`：

```yaml
---
title: Go 的 defer 使用与原理
date: 2022-11-02
categories:
  - go
tags:
  - Go
  - defer
---
```

`date` 决定永久链接（`/YYYY/MM/DD/<title>/`），文件名中的日期只是标记。

## 分类

固定分类：`go`、`mysql`、`redis`、`kafka`、`leetcode`，由两处定义：

1. `source/categories/<cat>/index.md`——分类列表页（`layout: category`）
2. `_config.typo.yml` 的 `categories_menu`——导航下拉项

`hexo-generator-category` 未安装，新分类必须手动建页，否则链接 404。分类页会自动列出 `categories: [<cat>]` 的文章。

## 发布

```bash
npx hexo clean
npx hexo generate      # 产物在 public/
npx hexo deploy        # 推送到 master
git add . && git commit -m "docs: ..." && git push origin source
```

GitHub Pages 源为 `master` 分支根目录，构建约需 1–3 分钟。验证新内容时加 `?v=<commit-sha>` 绕过缓存。
