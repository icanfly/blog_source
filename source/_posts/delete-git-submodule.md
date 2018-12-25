---
title: delete-git-submodule
tags:
  - git
categories:
  - 原创文章
  - 问题记录
originContent: ''
toc: false
author:
thumbnail:
blogexcerpt:
---

删除一个submodule

1. 删除 .gitsubmodule中对应submodule的条目

2. 删除 .git/config 中对应submodule的条目

3. 执行 git rm --cached {submodule_path}。注意，路径不要加后面的“/”。例如：你的submodule保存在 theme/maupassant/ 目录。执行命令为： git rm --cached theme/maupassant 