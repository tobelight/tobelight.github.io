---
title: 使用 Hexo 在 Github 上搭建博客
date: 2024-05-18 20:20:20
tags: hexo
---

作为一名 [INFJ-A](https://www.16personalities.com/ch/infj-%E4%BA%BA%E6%A0%BC)，我一直渴望拥有一个可以记录分享自己想法和作品的平台。经过一番探索，我最终选择了使用 Hexo 作为我的博客搭建工具。Hexo 以其简洁优雅的风格和强大的功能吸引了我，并帮助我顺利搭建了自己的博客。

在搭建博客的过程中，我遇到了一些需要解决的细节问题，也对一些默认配置进行了个性化调整。本文将记录我的搭建过程，并分享一些个人经验和建议，希望能对想要使用 Hexo 搭建博客的朋友有所帮助。

——**上面的内容由 [AI](https://gemini.google.com/) 生成，万事开头难，哈哈哈**

## 搭建博客

### 环境准备

1. [Node.js](https://nodejs.org/en/download/package-manager)
2. [hexo](https://hexo.io/zh-cn/docs/)

```bash
# 全局安装 hexo 命令工具
$ npm install -g hexo-cli
# 在空文件夹下初始化
$ hexo init
# 启动本地服务器，查看 Hello World
$ hexo server
```

### 使用 Github Action 自动部署到 Github Page

参考 [在 GitHub Pages 上部署 Hexo](https://hexo.io/zh-cn/docs/github-pages)

1. 在 Github 上创建 `<你的 GitHub 用户名>.github.io` 的仓库
2. 将 Hexo 文件夹中的文件 push 到储存库的默认分支
3. 在储存库中建立 `.github/workflows/pages.yml`，并填入下面内容

```yml .github/workflows/pages.yml
name: Pages

on:
  push:
    branches:
      - main # default branch

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          # If your repository depends on submodule, please see: https://github.com/actions/checkout
          submodules: recursive
      - name: Use Node.js 20
        uses: actions/setup-node@v4
        with:
          # Examples: 20, 18.19, >=16.20.2, lts/Iron, lts/Hydrogen, *, latest, current, node
          # Ref: https://github.com/actions/setup-node#supported-version-syntax
          node-version: "20"
      - name: Cache NPM dependencies
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    needs: build
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```
