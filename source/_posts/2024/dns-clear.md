---
# https://hexo.io/zh-cn/docs/front-matter
# layout: post
title: Windows 清除 DNS 缓存
date: 2024-05-21 22:20:00
updated: 2024-05-21 23:00:00
comments: true
tags: windows ipconfig dns
# categories:
# subtitle: 踩坑日志
# description: 使用 Hexo 搭建在 Github Page 搭建博客
# header-img: hello-world/banner.jpg
---

```bash
# 清空 dns
$ ipconfig /flushdns

# 查看 dns
$ ipconfig /displaydns
```
