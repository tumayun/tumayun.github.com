---
layout: post
title: "Rails 4 源码阅读之timestamps"
date: 2013-06-29 19:20
comments: true
categories: Rails Rails4
---

最近开始看 Rails 4 的源码,打算写一系列的 Rails 4 源码阅读的文章,这是第一篇.

我是因为想知道 Rails 4 里面的 timestamps 是在什么时候赋值或者更新的,然后我翻看了下 Rails 4 的 ActiveRecord 代码.

在[https://github.com/rails/rails/blob/master/activerecord/lib/active_record/timestamp.rb](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/timestamp.rb)里面有[create_record](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/timestamp.rb#L46)和[update_record](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/timestamp.rb#L60)方法,更新这些`timestamps`的操作就在这个里面了.

恩,应该是这样.`update_record`与`create_record`分别是创建和更新必掉的方法,所以如果有 timestamps 并且开启了`record_timestamps`就会去赋值或者更新这些字段了.

其实也可以从`save`开始阅读,`save`的过程可以简化成这样.

```ruby
# https://github.com/rails/rails/blob/master/activerecord/lib/active_record//connection_adapters/abstract/transaction.rb#L270
def save(*) #:nodoc:
  rollback_active_record_state! do
    with_transaction_returning_status { super }
  end
end
.
.
.
# https://github.com/rails/rails/blob/master/activerecord/lib/active_record/attribute_methods/dirty.rb#L31
def save(*)
  if status = super
    @previously_changed = changes
    @changed_attributes.clear
  end
  status
end
.
.
.
# https://github.com/rails/rails/blob/master/activerecord/lib/active_record/validations.rb#L56
def save(options={})
  perform_validations(options) ? super : false
end
.
.
.
# https://github.com/rails/rails/tree/master/activerecord/lib/activerecord/persistence.rb#L105
def save(*)
  create_or_update
rescue ActiveRecord::RecordInvalid
  false
end
.
.
.
# https://github.com/rails/rails/tree/master/activerecord/lib/activerecord/callbacks.rb#L298
def create_or_update #:nodoc:
  run_callbacks(:save) { super }
end
.
.
.
# https://github.com/rails/rails/tree/master/activerecord/lib/activerecord/persistence.rb#L464
def create_or_update
  raise ReadOnlyRecord if readonly?
  result = new_record? ? create_record : update_record
  result != false
end
.
.
.
# https://github.com/rails/rails/blob/master/activerecord/lib/active_record/timestamp.rb#L46
def create_record
  if self.record_timestamps
    current_time = current_time_from_proper_timezone

    all_timestamp_attributes.each do |column|
      if respond_to?(column) && respond_to?("#{column}=") && self.send(column).nil?
        write_attribute(column.to_s, current_time)
      end
    end
  end

  super
end
.
.
.
# https://github.com/rails/rails/blob/master/activerecord/lib/active_record/callbacks.rb#L302
def create_record #:nodoc:
  run_callbacks(:create) { super }
end
.
.
.
# https://github.com/rails/rails/blob/master/activerecord/lib/active_record/persistence.rb#L483
def create_record(attribute_names = @attributes.keys)
  attributes_values = arel_attributes_with_values_for_create(attribute_names)

  new_id = self.class.unscoped.insert attributes_values
  self.id ||= new_id if self.class.primary_key

  @new_record = false
  id
end
```

哇,就是这样的顺序,但是感觉有点复杂啊,`save`到处都是,通过`super`关键字逐层调用,就像一个`rack stack`一样!最主要的是我们要知道`save`的调用链是怎么样的,这个可以看`ActiveRecord::Base.ancestors`.

```ruby
puts ActiveRecord::Base.ancestors.join("\n")
ActiveRecord::Base
ActiveRecord::Core
ActiveRecord::Store
ActiveRecord::Serialization
ActiveModel::Serializers::Xml
ActiveModel::Serializers::JSON
ActiveModel::Serialization
ActiveRecord::Reflection
ActiveRecord::Transactions
ActiveRecord::Aggregations
ActiveRecord::NestedAttributes
ActiveRecord::AutosaveAssociation
ActiveModel::SecurePassword
ActiveRecord::Associations
ActiveRecord::Timestamp
ActiveModel::Validations::Callbacks
ActiveRecord::Callbacks
ActiveRecord::AttributeMethods::Serialization
ActiveRecord::AttributeMethods::Dirty
ActiveModel::Dirty
ActiveRecord::AttributeMethods::TimeZoneConversion
ActiveRecord::AttributeMethods::PrimaryKey
ActiveRecord::AttributeMethods::Query
ActiveRecord::AttributeMethods::BeforeTypeCast
ActiveRecord::AttributeMethods::Write
ActiveRecord::AttributeMethods::Read
ActiveRecord::AttributeMethods
ActiveModel::AttributeMethods
ActiveRecord::Locking::Pessimistic
ActiveRecord::Locking::Optimistic
ActiveRecord::CounterCache
ActiveRecord::Validations
ActiveModel::Validations::HelperMethods
ActiveSupport::Callbacks
ActiveModel::Validations
ActiveRecord::Integration
ActiveModel::Conversion
ActiveRecord::AttributeAssignment
ActiveModel::ForbiddenAttributesProtection
ActiveModel::DeprecatedMassAssignmentSecurity
ActiveRecord::Sanitization
ActiveRecord::Scoping::Named
ActiveRecord::Scoping::Default
ActiveRecord::Scoping
ActiveRecord::Inheritance
ActiveRecord::ModelSchema
ActiveRecord::ReadonlyAttributes
ActiveRecord::Persistence
Object
JSON::Ext::Generator::GeneratorMethods::Object
ActiveSupport::Dependencies::Loadable
PP::ObjectMixin
Kernel
BasicObject
```
这样就能知道调用链的情况了.当然还可以去`debugger`.

恩,`save`就是这样!`timestamps`又明白了,太棒了~!