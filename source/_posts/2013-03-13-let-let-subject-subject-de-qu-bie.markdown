---
layout: post
title: "let let! subject subject! 的区别"
date: 2013-03-13 13:56
comments: true
categories: rspec rails test
---
当我遇到不明白的事物的时候，我会先看表象，了解到表象后，再深入了解其内部。不能说这种方法好，但是对我来说还是挺有效的（年轻的coder）。当然，最好还是直接深入内部去了解，这样更省事省时。  
到正题，[let 与 let的说明](https://www.relishapp.com/rspec/rspec-core/v/2-13/docs/helper-methods/let-and-let! "let-and-let!")如下：

    Use let to define a memoized helper method. The value will be cached
    across multiple calls in the same example but not across examples.

    Note that let is lazy-evaluated: it is not evaluated until the first time
    the method it defines is invoked. You can use let! to force the method's
    invocation before each example.

其实 `let` `let!` 以及 `subject` 都是 memoized_helpers，都是通过委托来定义一个消息的接收方！好处就是可以让代码更精炼，提高可读性，减少重复。    
要比较三者间的区别，必须先聊聊`let`。

```ruby
def let(name, &block)
  # We have to pass the block directly to `define_method` to
  # allow it to use method constructs like `super` and `return`.
  MemoizedHelpers.module_for(self).define_method(name, &block)

  # Apply the memoization. The method has been defined in an ancestor
  # module so we can use `super` here to get the value.
  define_method(name) do
    __memoized.fetch(name) { |k| __memoized[k] = super(&nil) }
  end
end
```

首先定义一个`MemoizedHelpers`，就是`memoize method`，只要调用过，就会被缓存起来，下次调用还是返回上次相同的对象。
验证起来也很简单，可以看看两次调用结果的 `object_id`。  

```ruby
describe User do
  let(:user) { User.new }

  it 'test user' do
    puts user.object_id
    puts user.object_id
  end
end
```
输出结果：
    70186054302440
    70186054302440
###四者间的区别
首先上源码吧！ [let](https://github.com/rspec/rspec-core/blob/master/lib/rspec/core/memoized_helpers.rb#L177 "let") 源码上面已经贴了，
接下来看看 [let!](https://github.com/rspec/rspec-core/blob/master/lib/rspec/core/memoized_helpers.rb#L242 "let!")
[subject](https://github.com/rspec/rspec-core/blob/master/lib/rspec/core/memoized_helpers.rb#L276 "subject")
[subject!](https://github.com/rspec/rspec-core/blob/master/lib/rspec/core/memoized_helpers.rb#L342 "subject!")的源码。
####let!
```ruby
def let!(name, &block)
  let(name, &block)
    before { __send__(name) }
  end
```
####subject
```ruby
def subject(name=nil, &block)
  if name
    let(name, &block)
    subject { __send__ name }

    self::NamedSubjectPreventSuper.define_method(name) do
      raise NotImplementedError, "`super` in named subjects is not supported"
    end
  else
    let(:subject, &block)
  end
end
```

可以看得出，`let!`其实也是调用`let`，并且在`before each`调用，区别就是`let!`会在`before each`调用，而`let`不会，区别仅此而已！
`subject`就稍微的复杂了一点，首先如果`subject`指定了 name ，则会先用`let`生成一个叫 name 的 `memoize method`，然后再次用`let`生成一个叫 subject 的`memoize method`。这样当调用 subject 的时候，其实最终调用的是 name 这个`memoize method`。当`subject`没有指定 name 的时候，则生成用`let`直接生成一个名叫 subject 的`memoize method`。额，是不是有点乱？其实细细体味一下这个方法，哇塞，真的很棒，还这么精炼！用到了递归，哈哈！但是我又有点奇怪，为什么要多此一聚呢，而不直接用`let`。恩，看源码。  
```ruby
def should(matcher=nil, message=nil)
  RSpec::Expectations::PositiveExpectationHandler.handle_matcher(subject, matcher, message)
end
```
原来，`subject` 是用来配合 `should` 进行隐式调用的，在这里何为隐式调用？看源码里面的例子：
```ruby
describe CheckingAccount, "with $50" do
  subject { CheckingAccount.new(Money.new(50, :USD)) }
  it { should have_a_balance_of(Money.new(50, :USD)) }
  it { should_not be_overdrawn }
end
```
恩，`rspec`设计的就是这么巧妙！

####subject!

这个就不需要多说了，也就是比`subject`多了句`before each`。
```ruby
def subject!(name=nil, &block)
  subject(name, &block)
  before { subject }
end
```
###总结
其实什么时候用`let` `subject`，这个可以根据字面意思来，如果是测试主题，并且需要多次调用，就可有写成一个`subject`。而非测试主题，又需要多次调用，则可以用`let`！
