---
bg: "tools.jpg"
layout: post
title: "nginx keepalive && upstream timed out (110: Connection timed out) while connecting to upstream"
summary: ""
tags: ['nginx']
categories: nginx
---

昨天跟安全的同学对接7层转发，安全同学说前端nginx有很多"upstream timed out (110: Connection timed out)"错误，看了下提供的错误日志，有些请求是上传文件，有些则是普通的请求，查阅了下相关资料，应该有2个原因：一是因为高防接入层到我们的服务接入层走的是公网，所以网络的抖动是原因之一，简单的测了下连接时间，也证明了网络的不稳定性，二是让安全的同学开启了长连接keepalive

```shell
$while true;do dig +short your_domain;curl -so /dev/null -w %{time_namelookup}::%{time_connect}::%{time_starttransfer}::%{time_total}::%{speed_download}"\n" "https://your_domain";sleep 1;done
```

关于nginx的keepalive 与 upstream timed out

表面上看upstream time out是因为后端连接超时了，直观上我们可能希望增加proxy_connect_timeout的值，但是nginx文档中写明这个变量的默认值是60s，对于一个普通请求来说，这足够了，即便我们改大了，客户端的超时时间也远远小于这个值，单纯的修改这个变量对改善这种问题意义不大

在http2之前，nginx走的是http1.0协议，也就是说一个请求响应结束后，nginx会把连接断开，而当客户端有多个连续请求时，nginx不得不建立多个连接发送请求，而当采用长连接时，多个请求可以复用同一个连接，这样可以减少资源开销，节省请求时间

```nginx
proxy_http_version http/1.1;

```

