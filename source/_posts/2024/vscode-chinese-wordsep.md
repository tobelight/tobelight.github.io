---
# https://hexo.io/zh-cn/docs/front-matter
# layout: post
title: VSCode 双击选中，配置中文分隔符
subtitle: 在双击选中短语时，解决中文逗号被包含的问题
date: 2024-05-25 00:30:00
updated: 2024-05-25 00:30:00
comments: true
tags: ["VS Code", "wordsep", "tip"]
# categories:
# description:
# header-img: hello-world/banner.jpg
---

VS Code 默认情况的双击选中，会发生中文字符逗号和句号在短语中间的情况，用起来很不方便。

<!--more-->

我们可以配置 `editor.wordsep` 来达到令中文符号也被识别为分隔符的目的。

首先进入配置页面，输入 `editor.wordsep` 到过滤其中，查看配置选项。

![config](01.jpg)

在输入框中，末尾直接写入 `，。：《》` 等中文分隔符。

![word sep](02.jpg)
