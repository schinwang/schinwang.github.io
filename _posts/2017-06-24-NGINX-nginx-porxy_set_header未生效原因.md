---
bg: "tools.jpg"
layout: post
title: "nginx-porxy_set_header未生效原因"
summary: ""
tags: ['nginx']
categories: nginx
---

有时候会发现，nginx设置了某个变量，比如增加一个header变量(porxy_set_header)后并未有生效，原因是porxy_set_header的生效与否与其所在的Block有关

我们都知道，通常nginx配置文件会用到http, server 和 location Block，而proxy_set_header 在这3个Block下都是可以设置header头的，而此时就需要注意的是，nginx在生效变量时，会采取 ’就近原则‘ ，具体而言

* 如果在location中做了proxy_set_header， 在server或http Block中再添加header，对这个location就不生效了
* 同理如果location未做配置，而在server Block做了统一配置，那么在http中增加一个header头对这个server Block是不起作用的

所以对于通用的配置，我们都是在http Block中做统一配置，而如果有特殊配置的话，如果加在了location或者server Block，记得要继承http中的通用配置O(∩_∩)O哈！
