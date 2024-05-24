---
# https://hexo.io/zh-cn/docs/front-matter
# layout: post
title: Docker 默认的容器命名规则
# subtitle:
date: 2024-05-25 00:20:00
updated: 2024-05-25 00:20:00
comments: true
tags: ["docker", "arts", "arts-review"]
categories: ["arts", "review"]
# description:
# header-img: hello-world/banner.jpg
---

ARTS-R：Review，阅读并点评一篇英文文章

<!--more-->

文章原始地址：<https://pet2cattle.com/2022/08/docker-container-names-generator>

如果未使用 --name 设置容器默认名称，docker 会为您选择一个名称 —— 两个使用下划线连接的单词。

这是这个名字的生成逻辑：<https://github.com/moby/moby/blob/39f7b2b6d0156811d9683c6cb0743118ae516a11/pkg/namesgenerator/names-generator.go#L852-L863>

它会从两个列表中选取单词，一个形容词列表，一个名科学家和黑客的名字列表，如果遇到重复则会重新生成。

但是对于 woznia 是个例外。

```go
func GetRandomName(retry int) string {
begin:
  name := left[rand.Intn(len(left))] + "_" + right[rand.Intn(len(right))] //nolint:gosec // G404: Use of weak random number generator (math/rand instead of crypto/rand)
  if name == "boring_wozniak" /* Steve Wozniak is not boring */ {
    goto begin
  }

  if retry > 0 {
    name += strconv.Itoa(rand.Intn(10)) //nolint:gosec // G404: Use of weak random number generator (math/rand instead of crypto/rand)
  }
  return name
}
```
