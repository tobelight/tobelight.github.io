---
layout: post
title: 你的 git 该升级了！
subtitle: git 远程执行漏洞 —— cve-2024-32002
comments: true
tags:
  - "git"
  - "cve-2024-32002"
header-img: 2024/git-cve/banner.jpg
date: 2024-06-01 19:28:36
updated: 2024-06-01 19:28:36
---

执行以下命令：

`git clone --recursive https://github.com/tobelight/cve_2024_32002.git`

如果你的电脑打开了计算器，那么你的电脑就存在被远程执行代码的风险。

<!--more-->

## CVE-2024-32002

这个漏洞被报告在 <https://www.cve.org/CVERecord?id=CVE-2024-32002>

受影响的版本如下：

![version](version.jpg)

只要升级你的 git 大于以上版本，基本上就不会触发这个漏洞了。

顺便 windows 可能早就被打了[补丁](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2024-32002)，已经不会触发这个漏洞了。

## 原理研究

首先这个漏洞利用了 Windows，Mac 系统的大小写不敏感，如果你用的是 linux 以及其他大小写敏感的操作系统，就不会触发这个漏洞。

因此，要创建这个漏洞也得在大小写敏感的系统上。

### Git Hook

Git 在执行命令之后，可以自定义一些脚本自动执行，实现这个功能的就是 [Git Hooks](https://git-scm.com/book/zh/v2/自定义-Git-Git-钩子)。

这个功能通常可以用在代码审查，提交通知上面。比如执行 git commit 之后，执行命令发送一个邮件给管理员，检查代码的质量等。

这个漏洞利用的则是在 git clone 之后，自动切换到默认的分支时，触发的执行命令 `post-checkout` 脚本。

### 符号连接

我们知道 Windows 有快捷方式，linux，mac 上有 link 这样的功能，可以创建一个符号链接指向给定路径。

比如我可以创建一个符号链接 a，指向 `.git` 目录，这样当我向 a 路径下写入文件时，就可以把我想写入的文件写到 `.git` 路径下，当然这就是十分危险的了。

当然在 `git submodule add` 的时候，是不允许我们把文件写入到已经存在的路径的。比如我无法把 module 写入到刚创建的符号链接 a 中。

不过，我可以把 module 写入到没有创建的文件夹 A 中，这在大小写敏感的系统是成立的。

但是到了大小写不敏感的系统中，我们在拉取一个按照上面方法创建的，带有 git module 就会出现问题，系统会默认把文件成功写入到文件夹 a 中，即链接指向了 `.git` 目录，这就造成了一次危险的文件写入。

## 实验环节

知道原理就要动手实践了。

1. 在 github 中创建了 [hook 仓库](https://github.com/tobelight/cve_2024_32002_hook)，这里注意在要把脚本的权限加上可执行 `chmod +x post-checkout`。
2. 在 github 中创建了 [cve 仓库](https://github.com/tobelight/cve_2024_32002)，这里需要注意将 `.gitmodules` 文件中的 `submodule "A/modules/x"` 指向修改成 `submodule "x/y"`掉，这个指向决定了 submodule 的 `.git` 放在主目录的 `.git/modules` 路径下的位置。

## 实验结果

在 Mac 成功复现了远程代码执行漏洞，我的 git 版本是 2.39.3。
在 Windows 上没有复现漏洞，我的 Windows 系统的 git 版本是 `git version 2.41.0.windows.3` 不知道为什么默认设置了 `git config --global core.symlinks false`。

## 升级 Git

在升级 git 之后，重新执行命令，会发现在大小写不敏感的系统上，submodule 的文件夹覆盖掉了符号链接，也就不会触发上面的漏洞了。
