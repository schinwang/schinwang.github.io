---
bg: "tools.jpg"
layout: post
title: "nginx-判断header是否存在&&增加header"
summary: ""
tags: ['nginx']
categories: nginx
---

问题的背景是这样的，我们后端同学需要根据具体的请求协议(scheme)是http or https来作出相应，简单的我们可以加个X-Forwarded-Proto就好了，但是有时nginx有时会有2层（此时如果第一层处理https，proxy_pass也用https的话，会增加请求响应时间），所以我们需要判断请求是走了一层还是2层来设置X-Forwarded-Proto的值

我们不能直接写个if clause，判断头是否为空，然后加个proxy_set_header, 因为proxy_set_header 是不允许出现在if block中的

此时我们可以用nginx的map指令，map会根据传入的不同请求值给一个变量设置不同的值具体的语法参考[nginx map directive](http://nginx.org/en/docs/http/ngx_http_map_module.html)

```nginx
map $http_x_forwarded_proto $proto {
  default	$scheme;
  http		http;
  https		https;
}
```

> 这里有一点需要注意的是，任何的自定义header传到nginx时，需要将其转换为小写，短横线转换为下划线，并添加http_前缀，否则nginx会报没有该变量的错误

default表示默认值，如果X-Forwarded-Proto没有匹配到'http' 或'https', 也就是说如果没有定义这个header的话（意味着请求只走一层nginx），proto的值就等于$scheme(http请求头，标识请求协议是http还是https)，否则就等于X-Forwarded-Proto的值

然后我们可以增加一个请求头

```nginx
proxy_set_header X-Forwarded-Proto $proto
```

将proto的值赋给请求header： X-Forwarded-Proto

这样不管用户的请求是走一层nginx，还是连续走2层nginx，后端都能拿到正确的请求协议