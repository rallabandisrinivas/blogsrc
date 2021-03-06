---
title: linux1日志logrotate
date: 2018-11-04 12:12:28
tags:
- linux
---
#### linux1日志logrotate

- 引入管理日志工具logrotate
- 该工具需要配合crontab使用
<!--more-->
##### 1.1. 安装

- 大多数系统基本上已经安装

```bash
yum install logrotate crontabs -y
```
##### 1.2. 使用以及配置

- 主配置文件在/etc/logrotate.conf
- 从配置文件在中/etc/logrotate.d
- 配置示例详解

```bash
#对哪个日志进行分割
/var/log/nginx/access.log  {
#分割周期为月，可选daily/monthly/weekly/yearly
monthly
#一共存储6个归档
rotate 6
#日志分割期间忽略任何错误
missingok
#命名格式含有日期
dateext
#进行轮询后的日志进行压缩
compress
#对最近的归档不要压缩
delaycompress
#空日志不进行轮询
notifempty
#postrotate脚本在日志压缩后仅执行一次
sharedscripts
postrotate
    #发送-USR1信号，更改日志文件描述符
    [ -e /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
endscript
}
```
- cron定时任务已经自动创建，位置在/etc/cron.daily/logrotate，内容为

```bash
#!/bin/sh

/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
```
- 强制分割，以便测试
```bash
logrotate -vf /etc/logrotate.d/nginx
```

#### 2. 日志脚本

```bash
#!/bin/sh

d=`date +%Y%m%d`

cd /usr/local/apache-tomcat-8.5.24/logs
mkdir logbackup
cp catalina.out logbackup/catalina.out.${d}
echo "" > catalina.out
cd /usr/local/apache-tomcat-8.5.24/logs/logbackup
/bin/tar zcvf catalina.out.${d}.tar.gz catalina.out.${d}
rm -rf catalina.out.${d}
/bin/find  /usr/local/apache-tomcat-8.5.24/logs/logbackup -mtime +180  -exec rm -rf '{}' \;

#需要进行定时任务
echo "0 1 * * * /usr/local/apache-tomcat-8.5.24/cleanlog.sh" >> /var/spool/cron/root
#重启cron服务
service crond restart
```

#### 3. 日志切割nginx

```bash

#!/bin/bash
. /etc/profile
log_path='/kuaiyun/log/lb'
pid_path='/var/run/nginx.pid'
for afile in `find ${log_path} -name '*access.log' -type f`;do
  filename=`echo ${afile} |awk -F'/' '{print $NF}'`
  mv ${afile} ${log_path}/${filename}-`date +%Y%m%d`
done
for efile in `find ${log_path} -name '*error.log' -type f`;do
  filename=`echo ${efile} |awk -F'/' '{print $NF}'`
  mv ${efile} ${log_path}/${filename}-`date +%Y%m%d`
done
kill -USR1 `cat ${pid_path}`
find ${log_path} -mtime +30 -type f -exec rm {} \;

```
