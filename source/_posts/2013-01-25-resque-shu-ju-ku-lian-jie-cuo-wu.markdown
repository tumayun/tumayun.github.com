---
layout: post
title: "Resque 数据库链接错误"
date: 2013-01-25 14:07
comments: true
categories: Resque Mysql Rails
---
  &nbsp;&nbsp;最近项目用的 [Resque](https://github.com/defunkt/resque, 'Resque Github') 老是会有一些莫名其妙的问题，非常头疼！
```ruby
Mysql::Error: MySQL server has gone away: SHOW FIELDS FROM `deals`
```
  实在是没办法了，然后仔细的去阅读了下 [Resque](https://github.com/defunkt/resque, 'Resque Github') 的 [wiki](https://github.com/defunkt/resque/wiki, 'Resque wiki')，有种恍然大悟的感觉。
  原来我遇到的问题大家都遇到过，并且给出了解决方案，就拿 Resque 数据库链接错误来说，[Resque](https://github.com/defunkt/resque#mysqlerror-mysql-server-has-gone-away, 'Resque Github') 原作者已经有了推荐的解决方案，原文如下：  
  &nbsp;&nbsp;If your workers remain idle for too long they may lose their MySQL connection.
  If that happens we recommend using [this Gist](https://gist.github.com/238999).  
  &nbsp;&nbsp;然后我在项目 `config/initializers` 目录中的 `resque.rb` 文件中加入代码：
```ruby
Resque.after_fork do |job|
  ActiveRecord::Base.connection_handler.verify_active_connections!
end
```
问题就这样解决了！但是，引入一个 `Gem`，我连文档都没有仔细阅读，匆匆使用了事，现在想想真的很惭愧！
以后引入 `Gem` 最起码要将文档通读一边，有能力更应该通读源码！  
谨记！
