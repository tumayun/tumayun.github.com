---
layout: post
title: "Rails 技巧之 safe name"
date: 2013-06-23 23:51
comments: true
categories: rails ruby
---

当使用元编程动态生成代码时,可能会有一些不符合命名规范的名称,那么就可以用到下面的技巧,姑且称之为`safe name`.

以下出自 Rails 源码 [define_method_attribute](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/attribute_methods/read.rb#L50)

```ruby
# We want to generate the methods via module_eval rather than
# define_method, because define_method is slower on dispatch and
# uses more memory (because it creates a closure).
#
# But sometimes the database might return columns with
# characters that are not allowed in normal method names (like
# 'my_column(omg)'. So to work around this we first define with
# the __temp__ identifier, and then use alias method to rename
# it to what we want.
#
# We are also defining a constant to hold the frozen string of
# the attribute name. Using a constant means that we do not have
# to allocate an object on each call to the attribute method.
# Making it frozen means that it doesn't get duped when used to
# key the @attributes_cache in read_attribute.
def define_method_attribute(name)
  safe_name = name.unpack('h*').first
  generated_attribute_methods.module_eval <<-STR, __FILE__, __LINE__ + 1
  def __temp__#{safe_name}
  read_attribute(AttrNames::ATTR_#{safe_name}) { |n| missing_attribute(n, caller) }
  end
  alias_method #{name.inspect}, :__temp__#{safe_name}
  undef_method :__temp__#{safe_name}
  STR
end
```

`safe name`技巧的原理就是构造符合命名规范的名称(通过一些算法,像unpack,hash都可以),
  像这里的话是通过`unpack('h*')`生成一个字串`safe_name`.

  这个技巧真心很赞!!!
