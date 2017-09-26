---
bg: "superman.jpg"
layout: post
title: "supervisor延时启动程序"
summary: ""
tags: ['ops']
categories: ops
---

通过supervisor可以比较整洁、方便的管理同一台机器上的不同进程，实现对进程的监管及自动拉起。但有时RD希望某些程序能延时启动，supervisor是没有延迟启动参数的，我们可以在启动命令中增加sleep时间或者写个启动脚本来实现

```ini
[program:yourapp]
command = bash -c "sleep 60 && exec urcmd'
startsecs = 65 ; 
```

and then

```shell
$ supervisorctl -c your_config_file reload
```

- you need to use `exec` command, otherwise it will fork a subprogress from `sleep 60 && exec your command` and your progress will look like as the following

```shell
$ ps -ef|grep urcmd
work      1818  1698  0 17:35 ?        00:00:00 bash -c sleep 60 && urcmd
work      3872  1818  0 17:36 ?        00:00:00 urcmd
```

and then when you use `supervisorctl` to stop urApp, you will stop 1818 progress and leave 3872 an orphan progress

- recommend to change the startsecs to 5 more than the sleep secs, then when you start this app and check the status, it will show you it's starting

```shell
$ supervisorctl -c your_config_file status;echo;ps -ef|grep urcmd
urapp                          STARTING  
otherapp                       RUNNING   pid 13502, uptime 0:00:55

$ supervisorctl -c your_config_file status;echo;ps -ef|grep urcmd
urapp                          RUNNING   pid 13503, uptime 0:00:05
otherapp                       RUNNING   pid 13502, uptime 0:00:65
```

else if you set the value less than the sleep secs, when you start the app and check the status,you will get a running status, but it's still sleeping cmd before real executing

- when you change your configuration file, you need to use reload cmd or just restart your supervisord to make it work