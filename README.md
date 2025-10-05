# oyzg's note

个人技术博客，使用Hugo构建，部署在GitHub Pages上。

## 技术栈

- **静态网站生成器**: Hugo 0.147.4
- **主题**: PaperMod
- **部署平台**: GitHub Pages
- **域名**: https://oyzg.github.io/

## 本地开发

```bash
# 安装Hugo
brew install hugo

# 启动开发服务器
hugo server -D

# 构建网站
hugo --gc --minify
```

## 部署

网站通过GitHub Actions自动部署：

1. 推送代码到main分支
2. GitHub Actions自动构建Hugo网站
3. 部署到GitHub Pages

## 内容管理

### 发布新文章

```bash
# 1. 创建新文章
hugo new content/archives/文章标题.md

# 2. 编辑文章内容
# 修改前置元数据（title、categories、tags等）
# 编写文章内容
# 设置 draft: false 来发布文章

# 3. 本地预览
hugo server -D
# 访问 http://localhost:1313 查看效果

# 4. 构建网站
hugo --gc --minify

# 5. 提交并推送
git add .
git commit -m "Add new article: 文章标题"
git push origin main
```

### 文章结构

- 文章存放在 `content/archives/` 目录
- 使用Markdown格式编写
- 支持分类、标签、系列等组织方式

### 前置元数据格式

```yaml
---
title: "文章标题"
date: 2024-01-01T12:00:00+08:00
draft: false
categories: [分类1, 分类2]
series: [系列名称]
tags: [标签1, 标签2, 标签3]
summary: "文章摘要，会显示在首页和列表页"
---
```

## 许可证

MIT License
