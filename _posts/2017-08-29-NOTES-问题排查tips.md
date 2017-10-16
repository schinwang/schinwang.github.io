---
bg: "tree.jpeg"
layout: post
title: "web应用问题排查tips"
summary: ""
tags: ['notes']
categories: notes
---

#### 1 **快速查看服务器的应用及目录**

新登录到一个服务器，或者服务器上有很多应用时，怎么样找到每个应用的监听端口，及应用目录是很重要的

- 首先要判断是否docker服务，***docker ps***可以方便的查看是否是docker服务，是的话也可以列出当前在跑的容器服务
- 如果非docker容器的话，(如果是tcp web服务)可以通过***netstat   -lntp***, 或者***losf -iTCP -sTCP:listen***查看当前在监听请求的服务
- netstat 会显示监听服务的pid，运行进程一般在内存中分配资源，所以可以在/proc/${pid}/目录中查看进程命令，cwd(current work directory)等信息

#### 2 **日志总是最好的突破口**

- 如果有ELK等日志分析系统，通过filed查询可以方便的过滤出错误码及对应请求分布


- 否则，可以通过grep, awk等命令统计出自己想要的内容，比如

  - 过滤某个错误码

  ```shell
  grep -E "\s500\s" access_log
  ```

  - 统计最近一段时间请求数量（nginx为例）

  ```shell
  $awk '{print $7}' access_log|sort|uniq -c
  ```

#### 3 **链式跟踪，顺藤摸瓜**

很多服务一般都会有trace_id 方便跟踪请求，对某个请求调用链不熟悉的人来说，接入层日志（以nginx为例）是一个很好的切入点

- 从接入层可以过滤出问题请求，找到对应的upstream
- upstream一般为LVS/nginx, 这样可以定位到服务模块
- nginx日志中一般会有trace id用来标识一个请求，这样可以去应用日志查询到具体的问题

#### 4 **反向跟踪，必要时的抓包跟踪**

有时应用日志的报警定位需要找到某个请求具体的来源，

- 一般日志都会打印请求的地址


- 如果日志中没有打印的话，最好的方式就是抓包分析了

```shell
$tcpdump -nn -X dst host ${dst_ip} and dst port ${port} -w request.cap
```

如果请求量不是很大的话，可以找到请求的src

- 根据获取的src ip，即可反向跟踪问题

#### 5 app抓包利器 Charles

现在移动应用流行，好多web服务都会有mobile app，问题排查时，对app的请求与服务端响应分析可以通过Charles代理来实现

#### 6 请求模拟-postman

postman是一个很好的请求模拟器，支持cookie等需要登录操作的请求，是curl的一个升级版 ，交互方式十分友好，在问题复现及修复确认时，可以通过postman来模拟具体的请求