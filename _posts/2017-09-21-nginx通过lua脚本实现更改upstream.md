---
bg: "tools.jpg"
layout: post
title: "nginx-通过lua动态更改upstream"
summary: ""
tags: ['nginx']
categories: nginx
---

最近我们搭了一个consul服务，开发同学想要把supervisor的web管理集成到consul中，需要根据url中给定的ip地址动态的加载该机器上的supervisor管理界面，因为服务端机器都在VPC内部，办公网网络不可达，因此不能简单的rewrite url或者做个重定向来解决，因此需要通过反向代理的方式将请求转发给后端机器，并且要实现反向代理服务器的动态更改，由于逻辑比较复杂，所以就用lua脚本来实现了

**大致逻辑是这样的**

- upstream获取
  - 因为我们的consul服务通过域名访问，链到supervisor时，url中会有具体的upstream Ip，所以会先从url中看是否能match到ip，这样可以拿到upstream，并返回supervisor首页信息；
  - 当做tailf，start， stop 操作时，因为页面是从supervisor页链过去的，所以可以通过http_referer header来match upstream ip
- uri获取
  - 当拿到upstream后，从request_uri做字符串截取来拿到uri
- 然后一些css样式，image可以从本机获取

**具体实现如下**：

```nginx
location /supervisor/ {
    set $query_uri "";
  	#一开始从request_uri中把/supervisor/前缀去掉
    if ($request_uri ~* "/supervisor/(.*)") {
        set $query_uri $1;
    }
    set $upstreamhost "";
    rewrite_by_lua_block {
        ngx.var.upstreamhost = string.match(ngx.var.query_uri,"%d+.%d+.%d+.%d+")
        if ngx.var.upstreamhost == nil then
            ngx.var.upstreamhost = string.match(ngx.var.http_referer,"%d+.%d+.%d+.%d+")
        end
        index_start,index_end = string.find(ngx.var.query_uri,ngx.var.upstreamhost)
        if index_end ~= nil then
            tmp_uri = string.sub(ngx.var.query_uri,index_end+1)
            ngx.var.query_uri = tmp_uri
        end
    }
    proxy_pass http://$upstreamhost:8555/$query_uri;
    proxy_redirect http://$upstreamhost:8555 http://consul.url/supervisor/$upstreamhost;
}
location /supervisor/images/ {
    if ($request_uri ~* "/supervisor/(.*)") {
        proxy_pass http://127.0.0.1:8555/$1;
    }
}
location /supervisor/stylesheets/ {
    if ($request_uri ~* "/supervisor/(.*)") {
        proxy_pass http://127.0.0.1:8555/$1;
    }
}
```

因为在supervisor管理页面clear log的时候，服务器会返回Location header，所以通过proxy_redirect directive来做个重定向，让client（浏览器）还是走consul.server来发送请求，从而避免网络不可达问题

这样从本地办公网访问`http://consul.url/supervisor/10.10.10.12`就可以不用登陆服务器对服务进行启停及查看日志等操作了

**pprof集成**

同样再加pprof（go的性能调试工具）的时候，就可以复用这个逻辑了（因为go的pprof端口不一致，所以在url中给了ip:port, match的时候也多改成了"%d+.%d+.%d+.%d+:%d+"）

```nginx
location /debug/pprof/ {
    set $query_uri "";
    if ($request_uri ~* "/debug/pprof/(.*)") {
        set $query_uri $1;
    }
    set $upstreamhost "";
    rewrite_by_lua_block {
        ngx.var.upstreamhost = string.match(ngx.var.query_uri,"%d+.%d+.%d+.%d+:%d+")
        if ngx.var.upstreamhost == nil then
            ngx.var.upstreamhost = string.match(ngx.var.http_referer,"%d+.%d+.%d+.%d+:%d+")
        end
        index_start,index_end = string.find(ngx.var.query_uri,ngx.var.upstreamhost)
        if index_end ~= nil then
            tmp_uri = string.sub(ngx.var.query_uri,index_end+1)
            ngx.var.query_uri = tmp_uri
        end
    }
    proxy_pass http://$upstreamhost/debug/pprof/$query_uri;
}
```

这样开发从办公环境访问`http://consul.url/pprof/10.10.10.12:203`就可以对测试、开发环境的go程序进行简单的监控了

**改进: ip全部通过url获取**

有时开发希望通过命令行直接去看某台机器的pprof debug信息，为了方便粘贴浏览器的url，所以希望能把host地址包含进请求url，所以做了如下改进

```nginx
location /debug/pprof/ {
    set $query_uri "";
    if ($request_uri ~* "/debug/pprof/(.*)") {
        set $query_uri $1;
    }
    set $upstreamhost "";
    rewrite_by_lua_block {
        ngx.var.upstreamhost = string.match(ngx.var.query_uri,"%d+.%d+.%d+.%d+:%d+")
        if ngx.var.upstreamhost == nil then
            ngx.var.upstreamhost = string.match(ngx.var.http_referer,"%d+.%d+.%d+.%d+:%d+")
        end
        index_start,index_end = string.find(ngx.var.query_uri,ngx.var.upstreamhost)
        if index_end ~= nil then
            tmp_uri = string.sub(ngx.var.query_uri,index_end+2)
            ngx.var.query_uri = tmp_uri
        end
        local new_uri = "/debug/pprof/"..ngx.var.upstreamhost.."/"..ngx.var.query_uri
        --ngx.redirect(new_uri,301)
        ngx.req.set_uri(new_uri)
    }
    proxy_pass http://$upstreamhost/debug/pprof/$query_uri;
}
```

这样从`http://consul.url/pprof/10.10.10.12:203`返回的链接查看goroutine信息的时候，浏览器会跳转到`http://consul.url/pprof/10.10.10.12:203/goroutine?debug=1`



**关于多余代码**

如果set_uri了，那么下面这句代码看似就显得多余了

```nginx
if ngx.var.upstreamhost == nil then
    ngx.var.upstreamhost = string.match(ngx.var.http_referer,"%d+.%d+.%d+.%d+:%d+")
end
```

但是浏览器有个不好的地方就是会缓存重定向，而且单纯的disable cache不管用，因为之前都是跳到`http://consul.url/pprof/goroutine?debug=1`, 所以即使set_uri, 浏览器仍然不会跳转到新的url，***只有先301重定向破坏它的‘记忆’后才可以正常跳转***，shit~