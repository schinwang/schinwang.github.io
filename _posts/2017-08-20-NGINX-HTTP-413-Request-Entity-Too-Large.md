---
bg: "tools.jpg"
layout: post
title: "HTTP: 413 Request Entity Too Large"
summary: ""
tags: ['nginx']
categories: nginx
---

> 在进行文件上传的时候，有时会遇到413 Request Entity Too Large错误, 通常是由于nginx的client_max_body_size或者php的upload_max_filesize限制了上传文件大小

#####  client_max_body_size

```shell
Syntax:  client_max_body_size size;
Default: client_max_body_size 1m;
Context: http,server,location
```

Sets the maximum allowed size of the client request body, specified in the “Content-Length” request header field. If the size in a request exceeds the configured value, the 413 (Request Entity Too Large) error is returned to the client. Please be aware that browsers cannot correctly display this error. Setting `*size*` to 0 disables checking of client request body size.

client_max_body_site 默认只有1m大小，可以通过在http，server或者location模块里面添加该指令来设定上传大小限制，如果将该值设为0，那么nginx不会去检查上传大小的

##### upload_max_filesize

```shell
;;;;;;;;;;;;;;;;
; File Uploads ;
;;;;;;;;;;;;;;;;

; Whether to allow HTTP file uploads.
; http://php.net/file-uploads
file_uploads = On

; Temporary directory for HTTP uploaded files (will use system default if not
; specified).
; http://php.net/upload-tmp-dir
;upload_tmp_dir =

; Maximum allowed size for uploaded files.
; http://php.net/upload-max-filesize
upload_max_filesize = 2M

; Maximum number of files that can be uploaded via a single request
max_file_uploads = 20
```

file_uploads 用来打开或关闭文件上传功能

upload_max_filesize 用来限制上传文件的大小

max_file_uploads 用来限制同时上传文件的个数

##### 配置加载

php或者nginx都是在服务器启动的时候加载一次配置文件

nginx的配置加载可以通过发送HUP信号量优雅重启或者nginx -s reload来重新加载配置

php一般是通过fpm reload，可以通过发送USR2信号量来重新加载配置，另外yum/apt-get安装的一般有/etc/init.d/php-fpm脚本，src安装的，在安装包的sapi/fpm目录下有init.d.php-fpm脚本文件，通过修改prefix, 并将其拷贝到/etc/init.d/php-fpm 即可通过/etc/init.d/php-fpm reload 来重新加载配置文件

##### 附录

controlling nginx

| Signal    |                                          |
| --------- | ---------------------------------------- |
| TERM, INT | fast shutdown                            |
| QUIT      | graceful shutdown                        |
| HUP       | changing configuration, keeping up with a changed time zone (only for FreeBSD and Linux), starting new worker processes with a new configuration, graceful shutdown of old worker processes |
| USR1      | re-opening log files                     |
| USR2      | upgrading an executable file             |
| WINCH     | graceful shutdown of worker processes    |

controlling php-fpm

| Signal    |                                  |
| --------- | -------------------------------- |
| TERM, INT | fast shutdown                    |
| QUIT      | graceful shutdown                |
| USR1      | re-opening log files             |
| USR2      | Reload service php-fpm           |
| TERM      | Terminating php-fpm (force quit) |
