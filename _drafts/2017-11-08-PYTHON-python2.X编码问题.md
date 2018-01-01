---
bg: "tree.jpeg"
layout: post
title: "UnicodeEncodeError: 'ascii' codec can't encode characters in position 0-1"
summary: ""
tags: ['python']
categories: python
---

最近做kafka日志消费的时候，发现只要消息中有中文，就会报**UnicodeEncodeError: 'ascii' codec can't encode characters in position 0-1: ordinal not in range(128)**错误，所以重新学习了下python的编码，具体的可以参考[unicode-howto](https://docs.python.org/2.7/howto/unicode.html)

#### 首先要清楚Unicode的定义是什么

