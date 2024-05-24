---
# https://hexo.io/zh-cn/docs/front-matter
# layout: post
title: 初识 Hexo Theme
subtitle: Hexo 的 Theme 文件夹是如何运作的
date: 2024-05-24 23:30:00
updated: 2024-05-24 23:30:00
comments: true
tags: ["hexo", "hexo-theme"]
# categories:
# description:
# header-img: hello-world/banner.jpg
---

[hexo 英文文档](https://hexo.io/docs/themes)上提供了一个讲解 theme 的视频：

<!--more-->

<https://www.youtube.com/watch?v=5ROIU_9dYe4>

根据上面的视频学习。

hexo 的 themes 文件夹是从 `layout.ejs` 开始渲染的，所有的页面都会使用这个页面。

而 `index.ejs`, `about.ejs`, `post.ejs`, `page.ejs` 只是作为 body 内容被渲染在 `layout.ejs` 中。

## 以首页为例子，如何生成首页的 html

首先查看 `layout.ejs` 的代码，里面包含了整个页面最基础的元素 `<html>`, `<head>`, `<body>` 。我们可以看到代码中有一个 `body` 变量，在生成页面的时候，所有内容都经过它填充成为一个完整的页面。

```ejs layout.ejs
<!DOCTYPE html>
<html lang="en">

<%- partial('partials/head') %>

<!-- hack iOS CSS :active style -->
<body ontouchstart="">

    <%- partial('partials/nav') %>
    <%- partial('partials/search-page') %>

    <%- body %>

    <%- partial('partials/footer') %>


<!-- Image to hack wechat -->
<!-- <img src="/img/wechat.jpg" width="0" height="0" /> -->
<!-- Migrate from head to bottom, no longer block render and still work -->

</body>

</html>

```

接下来看看博客首页是如何生成的。首先博客首页会加载 `index.ejs` 的代码，这部分的代码在最开始的几行代码中指定了 `layout:page` 。这会导致该页面也是会作为 body 传递的，传递到指定的 `page.ejs` 中。

```ejs index.ejs
---
layout: page
---

<%- partial('partials/recent-posts') %>

<!-- Pager -->
<% if (page.total > 1) {%>
<ul class="pager">
  <% if (page.prev){ %>
      <li class="previous">
          <a href="<%- config.root %><%- page.prev_link %>">&larr; Newer Posts</a>
      </li>
  <% } %>
  <% if (page.next){ %>
      <li class="next">
          <a href="<%- config.root %><%- page.next_link %>">Older Posts &rarr;</a>
      </li>
  <% } %>
</ul>
<% } %>

```

然后查看 `page.ejs` 的代码，上面的 `index.ejs` 的内容作为 body 被渲染到这部分的代码中。然后这个 `page.ejs` 的代码再作为 body 渲染到我们最初看到的 `layout.ejs` 中，形成了一个页面。

```ejs page.ejs
---
layout: layout
---
......
  <%- body %>
......
```
