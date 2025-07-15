---
layout:     post
title:      Git 日常使用技巧
date:       2025-07-11 20:08:00
summary:    Git 日常使用技巧
image:	    "https://github.com/lina-not-linus"
tags:       Git
author:	    "shipujin"
excerpt_separator: <!--more-->
---

Git 日常使用技巧，提高生产力. <!--more-->


### Git usage tips and methods

1.更新第一次的commit（rebase -i 只能更新第2个commit）

```
git checkout --orphan new-master master
git commit -m "新的第一次commit内容"
```
