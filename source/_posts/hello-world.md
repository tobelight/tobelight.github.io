---
# https://hexo.io/zh-cn/docs/front-matter
# layout: post
title: 使用 Hexo 搭建博客
date: 2024-05-18 20:20:00
updated: 2024-05-19 21:30:00
comments: true
tags: hexo
# categories:
# subtitle: 踩坑日志
# description: 使用 Hexo 搭建在 Github Page 搭建博客
header-img: hello-world/banner.jpg
---

作为一名 [INFJ-A](https://www.16personalities.com/ch/infj-%E4%BA%BA%E6%A0%BC)，我一直渴望拥有一个可以记录分享自己想法和作品的平台。经过一番探索，我最终选择了使用 Hexo 作为我的博客搭建工具。Hexo 以其简洁优雅的风格和强大的功能吸引了我，并帮助我顺利搭建了自己的博客。

<!--more-->

在搭建博客的过程中，我遇到了一些需要解决的细节问题，也对一些默认配置进行了个性化调整。本文将记录我的搭建过程，并分享一些个人经验和建议，希望能对想要使用 Hexo 搭建博客的朋友有所帮助。

——**上面的内容由 [AI](https://gemini.google.com/) 生成，万事开头难，哈哈哈**

## 搭建博客

### 环境准备

1. [Node.js](https://nodejs.org/en/download/package-manager)
2. [Hexo](https://hexo.io/zh-cn/docs/)

```bash
# 全局安装 hexo 命令工具
$ npm install -g hexo-cli
# 在空文件夹下初始化
$ hexo init
# 启动本地服务器，查看 Hello World
$ hexo server
```

### 部署到 Github Page

参考 [在 GitHub Pages 上部署 Hexo](https://hexo.io/zh-cn/docs/github-pages)

1. 在 Github 上创建 `<你的 GitHub 用户名>.github.io` 的仓库
2. 将 Hexo 文件夹中的文件 push 到储存库的默认分支
3. 在储存库中建立 `.github/workflows/pages.yml`，并填入下面内容，再推送到 Github 上

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

## 个性配置

### 主题配置

这里我选择的是 [hux](https://huangxuan.me/) 的主题，但它是用 [jekyll](https://jekyllrb.com/) 构建的。
所以我们就在 Github 上找到了一个[把 hux 移植到 hexo 的主题](https://github.com/hhking/hexo-theme-huxo)。
然后把这个仓库 fork 到自己的仓库里面做定制化。

1. 为博客添加主题

   ```bash
   # 我把主题 fork 到了我自己的仓库 https://github.com/tobelight/hexo-theme-hux
   # 添加主题到我的 blog 中
   $ git submodule add https://github.com/tobelight/hexo-theme-hux themes/hexo-theme-hux
   ```

2. 在 config 中修改主题。

   ```yml _config.yml
   theme: hexo-theme-hux
   ```

3. 添加 less-renderer

   因为主题使用了 less 作为样式，所以需要添加依赖到 package.json 中。

   ```json package.json
   {
     "dependencies": {
       "hexo-renderer-less": "^4.0.0"
     }
   }
   ```

   ```bash
   # 安装依赖
   $ npm install
   # 清理之前生成的资源
   $ hexo clean
   # 执行 server 以本地查看效果
   $ hexo server
   ```

### 支持资源文件

参考 [Hexo 资源文件夹配置文档](https://hexo.io/zh-cn/docs/asset-folders)

```yml _config.yml
marked:
  prependRoot: true
  postAsset: true
```

### 生成 About 页面

```bash
# 生成 about 页面
$ hexo new page about
```

默认生成的 `index.md` 文件中没有指定 `layout: about`, 需要手动添加，
否则会导致 about 页面无法正常显示。

### 添加评论功能

参考 <https://giscus.app/zh-CN>
按照流程生成脚本，然后添加到 theme 中。

1. 在用户 Setting -> Application 中添加 giscus 应用，可指定权限
2. 在仓库的 Setting 中，找到 Discussions 并进行勾选
3. 在 [giscus](https://giscus.app/zh-CN) 页面中，获取最终脚本
4. 依据需求把脚本添加到 theme 的对应位置上

在这个页面上，任何人指定仓库就能获取到这段脚本，没有前置条件和权限控制。
这会导致任何人都可以把这个脚本用在任何一个网站上。
基于安全性考虑，我们还需要配置只有我们的网站才能使用这个脚本。

参考 <https://github.com/giscus/giscus/blob/main/ADVANCED-USAGE.md>

我们需要创建并配置 giscus.json 到仓库中，让 giscus 只允许我们配置的源使用。

### 字数统计

使用 <https://github.com/theme-next/hexo-symbols-count-time>

1. 安装字数统计的依赖 `npm install hexo-symbols-count-time`
2. 在 header 中添加以下脚本
   `<%- symbolsCount(page) %> words, <%- symbolsTime(page) %>`

## 问题排查

乱七八糟的问题整合。

### 部署 hexo 却出现 jekyll 错误

参考 issues: <https://github.com/orgs/community/discussions/106846>
参考 doc：<https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages#static-site-generators>

添加空文件 `.nojekyll` 到根目录。

![Alt](build-error.jpg)

### 配置支持资源文件时，markdown 语法没有生效

参考 issues：<https://github.com/hexojs/hexo-renderer-marked/issues/281>

hexo version: 7.2.0
hexo-renderer-marked: 6.3.0

目前 hexo-renderer-marked 没有成功支持 hexo `7.2.0`，需要降级到`7.1.1`。

```json package.json
{
  "hexo": {
    "version": "7.1.1"
  },
  "dependencies": {
    "hexo": "7.1.1"
  }
}
```

## 其他

编写 Hexo Theme 需要的属性以及方法都在官方文档中。
[支持的变量](https://hexo.io/zh-cn/docs/variables)
[支持的函数](https://hexo.io/zh-cn/docs/helpers)
