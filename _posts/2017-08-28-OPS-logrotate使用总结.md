---
bg: "superman.jpg"
layout: post
title: "logrotate 使用总结"
summary: ""
tags: ['ops']
categories: ops
---

​        关于线上日志切割，我始终认为比较好的实践方式有2种，一种是应用框架自带日志切割，可以让开发者根据需求按照文件大小或者按照时间自动切割日志(比如Java的log4j, 或者beego的日志方案)；另一种是实现重新打开日志的信号量(比如nginx、php-fpm的USER1)

​        最近团队引入了golang的zap框架(uber开源)，没有对日志做解决方案，所以只能借助logrotate的copytruncate功能实现日志切割

[基本配置](#config_example)

[常见问题](#common_problems)

[帮助手册](#man)

##### 基本配置 {#config_example}

基本的日志切割配置如下

```shell
/data/logs/module/*.log {
    copytruncate
    missingok
    notifempty
    nocompress
    maxage 10
    sharedscripts
    postrotate
        bash  /data/op_cron_jobs/logrotate_scripts/postrotate.sh /data/logs/module/ > /dev/null
    endscript
}

/data/logs/worker/stdout.log {
    su work work
    copytruncate
    missingok
    dateext
    dateformat .%Y%m%d
    notifempty
    nocompress
    postrotate
        bash /data/op_cron_jobs/logrotate_scripts/postrotate.sh /data/logs/worker/ "\."`date +"%Y%m%d"`
    endscript
}
```

因为logrotate默认是按天切割的，按小时切割的话，我们都希望切割后的日志名称能按时间戳来区分，所以用postrotate.sh完成文件的重命名

```shell
post_fix="\.1"
if [ $#'' == '2' ]
then
  post_fix=$2
fi

ls $1 |grep -E "$post_fix" |while read log
do
  new_log_name=$1"/"${log%%.*}"_log_"`date -d "-1 hour" +"%Y%m%d%H"`
  mv $1"/"$log $new_log_name
done
```

logrotate日志切分时，默认的后缀是.[1-9], 也可以用dateext生成以日期结尾的切割日志，当然因为没有小时，所以还是要用postrotate.sh加上小时

##### 常见问题 {#common_problems}

- 如果目标日志文件是root
  logrotate配置文件can be root or nonroot， 但必须通过sudo或以root账户来执行logrotate

- 如果目标日志文件非root
  ​      如果配置文件非root，需要以非root账户执行，否则容易遇到如下错误：

  ```shell
  Ignoring /data/logs/example.log because the file owner is wrong (should be root).
  ```

  ​      如果配置文件为root，需要以root用户或sudo执行，否则容易遇到以下错误

  ```shell
  error: error creating output file /var/lib/logrotate.status.tmp: Permission denied
  ```

  ​      centos7系统需要配置文件加 su user group, centos6 系统不需要,否则：

  ```shell
  error: skipping "/home/work/test/mate/clean.log" because parent directory has insecure permissions (It's world writable or writable by group which is not "root") Set "su" directive in config file to tell logrotate which user/group should be used for rotation.
  ```

综上，对于copytruncate模式的logrotate，将logrotate配置文件置为root归属，然后用sudo命令或root账户执行，可以避免很多问题

##### [附:logrotate帮助手册](https://linux.die.net/man/8/logrotate) {#man}