---
layout: post
title: "日志分割"
date: 2014-09-03 16:30:42 +0800
comments: true
categories: Rails logrotate
---
日志做分割是必须的，不然服务一直跑下去，某一天你会发现磁盘满了！下面就介绍下 logrotate 做日志分割的过程。

首先得安装 logrotate 。
```
sudo apt-get install logrotate
```

修改 logrotate 的配置文件。
```
sudo vim /etc/logrotate.conf

/home/www/webapp/iconfont/shared/log/*.log {
  daily         #按天分割
  rotate 30     #分割的文件最多有 30 份，将删除较早的文件
  missingok     #如果指定的文件不存在，不报错，继续下一个
  dateext       #用日期作为分割后的文件后缀
  compress      #对分割后的日志文件进行压缩
  delaycompress #不压缩前一个(previous)分割的文件（需要与compress一起用）
  copytruncate  #清空原有文件，而不是创建一个新文件
}
```

这样的话每天都会自动做日志分割，当然也可以强制执行。
```
sudo -vf logrotate -f /etc/logrotate.conf
```

想了解 logrotate 的配置文件该怎么写，可以看 [http://linuxers.org/howto/howto-use-logrotate-manage-log-files](http://linuxers.org/howto/howto-use-logrotate-manage-log-files)。
