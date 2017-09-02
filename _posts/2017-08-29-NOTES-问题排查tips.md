---
bg: "tree.jpeg"
layout: post
title: "问题排查tips"
summary: ""
tags: ['notes']
categories: notes
---



通过应用日志过滤请求，查看访问量异常请求, 以nginx日志为例

```shell
awk '{print $7}' access_log|sort|uniq -c
```

