# winlee.github.io

个人技术博客，基于 Hexo + [hexo-theme-keep](https://github.com/XPoet/hexo-theme-keep)，发布在 <https://winlee.github.io>。

## 分支说明

| 分支 | 内容 |
|---|---|
| `source` | Hexo 项目源码（配置、文章、主题、脚手架） |
| `master` | `hexo deploy` 生成的静态站点，GitHub Pages 从这里发布 |

不要手改 `master`，也不要提交 `public/`、`db.json`、`.deploy_git/`——它们都是生成物，已在 `.gitignore` 中。

## 目录结构

```text
_config.yml          # Hexo 主配置（站点信息、permalink、deploy）
_config.keep.yml     # Keep 主题配置覆盖（品牌、菜单、首屏、文章功能）
scaffolds/           # 新建文章/页面的模板
source/
  _posts/            # 文章 markdown
  categories/        # 分类索引页；分类详情由 hexo-generator-category 生成
  tags/              # 标签索引页；标签详情由 hexo-generator-tag 生成
  about/             # 关于页
  favicon.svg        # 站点自定义标识
  .nojekyll          # 跳过 Jekyll（随产物复制到 public/）
node_modules/hexo-theme-keep/ # Keep 官方主题 npm 包（不提交）
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

## 分类与标签

文章 front matter 中通过 `categories` 和 `tags` 设置分类与标签：

```yaml
categories:
  - go
tags:
  - Go
```

Keep 主题按 `source/categories/index.md`、`source/tags/index.md` 提供索引页；`hexo-generator-category` 和 `hexo-generator-tag` 自动生成具体分类与标签页面。新增分类不需要手工建分类目录页。

## 发布

```bash
npx hexo clean
npx hexo generate      # 产物在 public/
npx hexo deploy        # 推送到 master
git add . && git commit -m "docs: ..." && git push origin source
```

GitHub Pages 源为 `master` 分支根目录，构建约需 1–3 分钟。验证新内容时加 `?v=<commit-sha>` 绕过缓存。
