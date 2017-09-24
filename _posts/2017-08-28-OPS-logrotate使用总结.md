---
bg: "superman.jpg"
layout: post
title: "ops-logrotate使用总结"
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
/data/log/*/*.log {
    #if the logs' owner/group not root:root, need su directive to specify the owner/group'
    copytruncate
    missingok
    notifempty
    nocompress
    maxage 10
    sharedscripts
    postrotate
        bash  /data/op_cron_jobs/logrotate_scripts/postrotate.sh /data/log > /dev/null
    endscript
}
/path/to/worker/stdout.log {
    su work work
    copytruncate
    missingok
    dateext
    dateformat .%Y%m%d
    notifempty
    nocompress
    postrotate
        bash /data/op_cron_jobs/logrotate_scripts/postrotate.sh /path/to/worker/ "\."`date +"%Y%m%d"`
    endscript
}
```

因为logrotate默认是按天切割的，按小时切割的话，我们都希望切割后的日志名称能按时间戳来区分，所以用postrotate.sh完成文件的重命名，同时按天做了文件夹归档

```shell
post_fix="\.1"
if [ ""$# == '2' ]
then
    post_fix=$2
fi

archive() {
    if [ -d $1 ]
    then
    	#这里需要-1小时，避免每天23点的日志放到第二天的文件夹里
        today=`date -d "-1 hour" +%Y%m%d`
        archive_path=$1
        archive_dir=${today}_rotate
        test -d ${archive_path}/${archive_dir} || (mkdir -p ${archive_path}/${archive_dir} && chown -R work:work ${archive_path}/${archive_dir})
        ls $archive_path |grep -E "${post_fix}$" |while read log
        do
            new_log_name=${archive_path}/${archive_dir}/${log%%.*}"_log_"`date -d "-1 hour" +"%Y%m%d%H"`
            mv ${archive_path}/${log} ${new_log_name}
            find $archive_path -name "*_rotate" -mtime +14 -exec rm -rf {} \;
        done
    fi
}

path=$1
#首先确认是否有子文件夹（对应上面的/data/log/*/*.log），并对子文件夹遍历
is_here=`ls $path|grep -E "${post_fix}$"|wc -l`
if [ $is_here"" == "0" ]
then
    ls $path | while read subdir
    do
        echo $subdir
        archive ${path}/${subdir##*/}
    done
else
    archive $path
fi
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

- logrotate调试
  logrotate的-d命令会详细的给出执行过程，对于调试很有帮助

综上，对于copytruncate模式的logrotate，将logrotate配置文件置为root归属，然后用sudo命令或root账户执行，可以避免很多问题

##### [附:logrotate帮助手册](https://linux.die.net/man/8/logrotate) {#man}