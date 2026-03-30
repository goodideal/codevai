---
title: "Hugo + Stack 主题部署实战：从 XML 地狱到正确姿态"
description: "完整的 Hugo Stack 主题配置、troubleshooting 与 GitHub Actions 部署指南"
date: 2026-02-28
categories:
  - 博客部署
  - Hugo 配置
tags:
  - hugo
  - stack-theme
  - github-pages
  - ci-cd
slug: hugo-stack-deployment
---

## 问题：Hugo 为什么只吐 XML？

某天早上，我启动 Hugo 服务，访问博客首页——结果看到的是一堆 RSS XML，而不是优雅的 HTML 页面。

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<rss version="2.0">
  <channel>
    <title>My Site</title>
    ...
  </channel>
</rss>
```

## 根因分析

Hugo 有一个隐秘的行为：**如果主模板找不到，或配置错误，它会降级至"仅生成 RSS"模式**。

这通常由以下触发：

1. **baseURL 配置错误**
2. **主题模板缺失**（submodule 没拉全）
3. **defaultContentLanguage 与 languages.toml 不匹配** ⚠️ 这就是我们的坑

## 坑点 1：languages.toml 语言键不一致

Stack 主题要求在 `languages.toml` 中声明每一个语言：

```toml
# config/_default/languages.toml
[zh]
title = "codevai"
weight = 1

[en]
title = "codevai"
weight = 2
```

但我最初设置的是：

```toml
defaultContentLanguage = "zh-cn"  # 不存在的键！
```

Hugo 在匹配时找不到 `zh-cn` 对应的配置，所以直接放弃了 HTML 渲染，改为生成 RSS。

**解决**：改为 `defaultContentLanguage = "zh"`，让键与 `languages.toml` 中的 `[zh]` 对齐。

## 坑点 2：Hugo 版本与主题兼容性

Stack 主题在生成目录时需要用到 `hash` 函数，这个函数在 Hugo 0.149.0 以下不存在。

如果 GitHub Actions 中的 Hugo 版本太低：

```bash
# ❌ 不行（旧版本没有 hash）
- uses: peaceiris/actions-hugo@v2
  with:
    hugo-version: '0.148.0'  

# ✅ 必须升至 0.154.0+
- uses: peaceiris/actions-hugo@v2
  with:
    hugo-version: '0.154.0'
```

构建时会报错：`function "hash" not found`，导致 HTML 生成失败。

## 坑点 3：Git Submodule 拉取不完整

Stack 主题被 fork 后加入了自定义样式，但如果 CI 没有正确拉 submodule，构建会缺少主题文件：

```bash
# ❌ 不完整
git checkout --recurse-submodules

# ✅ 完整（包含嵌套子模块）
git checkout --recurse-submodules
git submodule update --init --recursive
```

对应到 GitHub Actions：

```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive  # ← 这一行很关键
```

## 坑点 4：首页与分类页的 _index.md 缺失

Stack 主题需要明确的首页定义：

```bash
content/
├── _index.md          # 首页内容
├── _index.zh.md       # 中文首页
├── posts/
│   └── _index.md      # posts 分类页
├── about/
│   └── _index.md      # about 分类页
└── search/
    └── _index.md      # 搜索页（重要！）
```

如果这些 `_index.md` 缺失，对应的页面就会返回 404 或空白。

## 完整解决方案

### 1. 正确的 languages.toml

```toml
# config/_default/languages.toml
[zh]
title = "codevai"
weight = 1
languageName = "中文"

[en]
title = "codevai"
weight = 2
languageName = "English"
```

### 2. GitHub Actions 工作流

```yaml
name: Deploy Hugo

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive  # ← 关键
      
      - uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.154.0'  # ← 必须 0.154.0+
      
      - name: Build
        run: hugo --minify
      
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          cname: goodideal.github.io
```

### 3. SSH Deploy Key 管理（避免冲突）

```bash
# 为博客 repo 生成专用 key
ssh-keygen -t ed25519 -f ~/.ssh/id_blog -N ""

# 在 ~/.ssh/config 中配置
Host github.com-blog
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_blog
  IdentitiesOnly yes
```

## 教训

1. **Hugo 的"降级至 RSS"模式是静默的**——没有明确的错误提示，很容易被错过。
2. **defaultContentLanguage 必须与 languages.toml 中的某个键精确匹配**，不能自作聪明用 `zh-cn`。
3. **GitHub Actions 中的 Hugo 版本最好保持最新**，避免兼容性坑。
4. **submodules: recursive 一定要加**，否则自定义主题无法完全加载。

---

**相关资源**：  
- [Hugo 官方文档](https://gohugo.io/documentation/)  
- [Stack 主题文档](https://stack.jimmycai.com/)
