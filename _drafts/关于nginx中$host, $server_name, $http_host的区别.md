## 关于nginx中\$host, \$server_name, $http_host的区别

`$server_name`

Server names are defined using the [server_name](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) directive and determine which [server](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) block is used for a given request

server_name 是nginx配置文件中的一个指令（directive), 支持通配符，用来匹配请求到来时，该用哪个server block来处理请求

`$host`

in this order of precedence: host name from the request line, or host name from the “Host” request header field, or the server name matching a request

$host是nginx配置文件中的一个变量，其值按如下顺序来确定，如果request line (写过socket的同学应该了解建立TCP连接后会发送类似'GET / HTTP 1.1'类似的就是请求行)中有的话就是这个值，否则从请求头中的Host值获取(http/1.0中，http_host有可能为空哦，http/1.1，host必须存在），否则从匹配到server_name（server_name有可能是*.xxxx.com这种呢，如果用这种值proxy_pass到某个域名的话，会报400哦)来确定

`$http_host`

nginx官方并没有对其的解释，但之前在用nginx做代理的时候，直接proxy_pass到目标网址会返回400，发现是因为$host的值为*.xxx.com(取了server_name?), 于是proxy_set_header host \$http_host 临时来解决

至于原因，暂未有深入了解，原谅我的囫囵吞枣O(∩_∩)O~