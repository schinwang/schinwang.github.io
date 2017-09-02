---
bg: "meteor.jpg"
layout: post
title: "docker使用supervisor的一些问题"
summary: ""
tags: ['docker']
categories: docker
author: chason
---

docker 中管理(多)进程的时候, 如何在关闭容器的时候优雅关闭容器中的进程是一个需要解决的问题，很多人会选择使用supervisord，下面是其中一个配置示例,需要注意的是supervisord进程需要以no daemon形式存在

{% highlight shell %}
[supervisord]
http_port=/var/tmp/supervisor.sock          ; (default is to run a UNIX domain socket server)
;http_port=127.0.0.1:9001                   ; (alternately, ip_address:port specifies AF_INET)
;sockchmod=0700                             ; AF_UNIX socketmode (AF_INET ignore, default 0700)
;sockchown=work.work                        ; AF_UNIX socket uid.gid owner (AF_INET ignores)
;umask=022                                  ; (process file creation umask;default 022)
logfile=/var/log/supervisor/supervisord.log ; (main log file;default $CWD/supervisord.log)
logfile_maxbytes=50MB                       ; (max main logfile bytes b4 rotation;default 50MB)
logfile_backups=10                          ; (num of main logfile rotation backups;default 10)
loglevel=warn                               ; (logging level;default info; others: debug,warn)
pidfile=/var/run/supervisord.pid            ; (supervisord pidfile;default supervisord.pid)
nodaemon=true                               ; (start in foreground if true;default false)
minfds=1024                                 ; (min. avail startup file descriptors;default 1024)
minprocs=200                                ; (min. avail process descriptors;default 200)

[supervisorctl]
serverurl=unix:///var/tmp/supervisor.sock   ; use a unix:// URL  for a unix socket
;serverurl=http://127.0.0.1:9001            ; use an http:// url to specify an inet socket
;prompt=mysupervisor                        ; cmd line prompt (default "supervisor")
; The below sample program section shows all possible program subsection values,
; create one or more 'real' program: sections to be able to control them under
; supervisor.

[program:nginx]
command=bash -c "/usr/bin/openresty -p /usr/local/nginx/"
log_stdout=true                                                                    ; if true, log program stdout (default true)
log_stderr=true                                                                    ; if true, log program stderr (def false)
logfile=/data/log/nginx_progress.log                                 ; child log path, use NONE for none; stopsignal=TERM
default AUTO

[program:php-fpm]
user=work                                                                          ; setuid to this UNIX account to run the program
command='/data/php/sbin/php-fpm'
autostart=true                                                                     ; start at supervisord start (default: true)
autorestart=true                                                                   ; retstart at unexpected quit (default: true)
stopsignal=TERM                                                                    ; signal used to kill process (default TERM)
stopwaitsecs=10                                                                    ; max num secs to wait before SIGKILL (default 10)
logfile=/data/log/php_progress.log                                   ; child log path, use NONE for none; default AUTO

{% endhighlight %}

但是使用的过程中有几个问题

#### 1. 我们在容器中执行supervisorctl stop/start/tail/status命令的时候会碰到

{% highlight shell %}

unix:///var/tmp/supervisor.sock refused

{% endhighlight %}

的错误，这是一个关于[overlayfs](https://github.com/Supervisor/supervisor/issues/654)的问题([So long as that volume's underlying filesystem is 'normal' for linux and will permit the creation of working sockets](https://github.com/analytically/docker-overlayfs-bug))When using device mapper, it works fine.可以在容器中启动的时候添加

{% highlight shell %}
--tmpfs /var/tmp/
{% endhighlight %}

来解决

#### 2. supervisorctl stop \<process\>的时候会hang住

{% highlight shell %}
$ sudo supervisorctl stop nginx
^CTraceback (most recent call last):
File "/usr/bin/supervisorctl", line 6, in \<module\>
  main()
File "/usr/lib/python2.6/site-packages/supervisor/supervisorctl.py", line 598, in main
  c.onecmd(" ".join(options.args))
File "/usr/lib/python2.6/site-packages/supervisor/supervisorctl.py", line 86, in onecmd
  return func(arg)
File "/usr/lib/python2.6/site-packages/supervisor/supervisorctl.py", line 433, in do_stop
   result = supervisor.stopProcess(processname)
File "/usr/lib64/python2.6/xmlrpclib.py", line 1199, in \__call\__
  return self.\__send(self.\__name, args)
File "/usr/lib64/python2.6/xmlrpclib.py", line 1489, in __request__
  \__verbose=self.\__verbose
File "/usr/lib/python2.6/site-packages/supervisor/options.py", line 1309, in request
  errcode, errmsg, headers = h.getreply()
File "/usr/lib64/python2.6/httplib.py", line 1064, in getreply
  response = self._conn.getresponse()
File "/usr/lib64/python2.6/httplib.py", line 990, in getresponse
  response.begin()
File "/usr/lib64/python2.6/httplib.py", line 391, in begin
  version, status, reason = self._read_status()
File "/usr/lib64/python2.6/httplib.py", line 349, in _read_status
  line = self.fp.readline()
File "/usr/lib64/python2.6/socket.py", line 433, in readline
  data = recv(1)
KeyboardInterrupt

{% endhighlight %}

这也是个[已知的问题](https://github.com/Supervisor/supervisor/issues/131)，用supervisor-2.1*版本会出现这个问题, 建议升级到最新的版本
可以使用如下命令临时解决下（虽然不能解决本质问题）

```shell
timeout 3 supervisorctl stop nginx
```



#### 另外一种pid=1进程实现

docker run/start 执行启动后，启动命令cmd会启动一个pid为1的进程，docker stop的时候会向该进程发送一个TERM信号，但在多进程的容器中或者希望接受其他信号的进程容器中，希望有一个init进程能够负责处理该TERM信号实现对目标进程的优雅管理，supervisord就是这样的一种存在

当然，也可以根据其原理，写一个简单的shell脚本

{% highlight shell %}
\#!/bin/sh
sudo service rsyslog start
sudo /etc/init.d/crond start
/path-to/php/sbin/php-fpm start
sh /path-to/webserver/loadnginx.sh start
trap "sh /path-to/webserver/loadnginx.sh stop; exit" TERM
while true
do
  sleep 5
done
{% endhighlight %}

利用trap命令来捕捉TERM信号并作出对应的action
