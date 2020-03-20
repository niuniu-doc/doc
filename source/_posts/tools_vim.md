---
title: vim
date: 2020-03-20
categories:
  - Tools
tags:
  - Tools
---
* vim中文乱码 ` set fileencodings=utf-8`
* 十六进制显示 `:!xxd`
* 十六进制显示, n个字节为一组 `:!xxd -g n`
* n,m 行复制到x行之后 `:n,m co x`
* n,m 行移动到x行之后 `:n,m m x`
* 一批id、组装成SQL
`:%s/^/select field from table where id=/g`
* vim匹配指定模式删除 `:g/pattern/d`
* vim删除非指定模式的行 `:v/pattern/d` or `:g!/pattern/d`
* 包含指定字符的个数 `:%s/pattern//gn`
* 忽略大小写查找 `/