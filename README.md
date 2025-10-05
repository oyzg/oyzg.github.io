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

- 文章存放在 `content/archives/` 目录
- 使用Markdown格式编写
- 支持分类、标签、系列等组织方式

## 许可证

MIT License
