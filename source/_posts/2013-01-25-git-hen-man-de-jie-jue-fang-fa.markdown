---
layout: post
title: "Git 很慢的解决方法"
date: 2013-01-25 00:32
comments: true
categories: Git
---
  修改文件`/etc/ssh/ssh_config`, 添加如下代码：
```ruby
GSSAPIAuthentication no
```
