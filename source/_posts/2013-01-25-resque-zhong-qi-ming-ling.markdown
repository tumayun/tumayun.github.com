---
layout: post
title: "Resque 重启命令"
date: 2013-01-25 00:46
comments: true
categories: Resque God
---
  &nbsp;&nbsp;现在 Resque 都应该加上 [god](http://godrb.com "Ruby process monito") 监控，不然 Resque worker 进程很容易就死掉！
  [god](http://godrb.com "Ruby process monito") 在 worker 进程挂掉后会自动重启，
  如果项目更新，需要手动重启所有的 worker，可以 kill 掉所有的 worker 进程(千万别 kill -9，最好是 kill -s QUIT，这样会在当前 job 完成后再退出)，
  方法如下：
```ruby
ps -e -o pid,command | grep [r]esque-[0-9] | cut -d ' ' -f 1 | xargs  kill -s QUIT
```
