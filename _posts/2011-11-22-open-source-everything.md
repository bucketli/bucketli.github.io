---
layout: post
title: 拉取mysql binlog停在错误位点的解决方法
---

{{ page.title }}
================

<p class="meta">27 Sep 2013 - 杭州</p>

1. 登陆mysql
2. 执行下面sql获取位点附近的binlog记录
```
show binlog events in 'mysql-bin.002334' from 0482929187 limit 10
```
3. 选取对应位点的binlog
