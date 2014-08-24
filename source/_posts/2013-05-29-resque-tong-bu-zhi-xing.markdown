---
layout: post
title: "Resque 同步执行"
date: 2013-05-29 01:16
comments: true
categories: Resque
---

### 背景

测试的时候,有个方法里面调用了`Resque.enqueue`,我觉得不好测试.
可能有人会说只需要测试是否调用了`Resque.enqueue`,或者只需要测试`redis`里面是否有这个队列.
但是我觉得最好还是要测试到后台任务的执行结果,因为后台任务的最终结果可能就是这个方法的关键之处,是这个方法的主体.所以最好还是能去测试后台任务的结果(当然,视代价而定)!~

好了,回到正题.怎么去测试`Resque`任务的执行结果呢?额,开着后台`worker`?然后在测试代码里面`sleep 10`等待后台执行?我擦,太二了!

翻看了下[Resque](https://github.com/resque/resque)的源码,发现了个好东西[inline](https://github.com/resque/resque/blob/master/lib/resque.rb#L119).

<pre>
# If 'inline' is true Resque will call #perform method inline
# without queuing it into Redis and without any Resque callbacks.
# The 'inline' is false Resque jobs will be put in queue regularly.
</pre>

大意就是

<pre>
如果 inline 为 true, Resque 任务会直接执行不需要放到 redis 队列中，并且不执行任何 callbacks。 inline 是 false, Resque
任务将放在 redis 队列里面等待执行。
</pre>

恩,nice!这个就是我想要的,同步执行!

### 源码分析 inline

首先要看压队列的方法 [Resque#enqueue](https://github.com/resque/resque/blob/master/lib/resque.rb#L245).

```ruby
def enqueue(klass, *args)
  enqueue_to(queue_from_class(klass), klass, *args)
end
```
这个方法是将`job`压入`redis`队列.再看看 [queue_from_class](https://github.com/resque/resque/blob/master/lib/resque.rb#L336)方法.

```ruby
def queue_from_class(klass)
  if klass.instance_variable_defined?(:@queue)
    klass.instance_variable_get(:@queue)
  else
    (klass.respond_to?(:queue) and klass.queue)
  end
end
```
这个方法是用来获取`job`对应的队列名的.

再回到`enqueue`方法,看看 [enqueue_to](https://github.com/resque/resque/blob/master/lib/resque.rb#L258) 方法.

```ruby
def enqueue_to(queue, klass, *args)
  validate(klass, queue)
  # Perform before_enqueue hooks. Don't perform enqueue if any hook returns false
  before_hooks = Plugin.before_enqueue_hooks(klass).collect do |hook|
    klass.send(hook, *args)
  end
  return nil if before_hooks.any? { |result| result == false }

  Job.create(queue, klass, *args)

  Plugin.after_enqueue_hooks(klass).each do |hook|
    klass.send(hook, *args)
  end

  return true
end
```
这个方法是去验证`job`,并且执行`hooks`,创建`job`.
先看看 [validate](https://github.com/resque/resque/blob/master/lib/resque.rb#L349).

```ruby
def validate(klass, queue = nil)
  queue ||= queue_from_class(klass)

  unless queue
    raise NoQueueError.new("Jobs must be placed onto a queue.")
  end

  if klass.to_s.empty?
    raise NoClassError.new("Jobs must be given a class.")
  end
end
```
这个方法是去校验是否有队列名以及`job class`.

再看看 [Plugin.before_enqueue_hooks](https://github.com/resque/resque/blob/master/lib/resque/plugin.rb#L64).

```ruby
# Given an object, returns a list `before_enqueue` hook names.
def before_enqueue_hooks(job)
  get_hook_names(job, 'before_enqueue')
end  
return nil if before_hooks.any? { |result| result == false }
```
这个方法是获取`before_enqueue hook`的列表.然后

```ruby
before_hooks = Plugin.before_enqueue_hooks(klass).collect do |hook|
  klass.send(hook, *args)
end
```
这段代码会去执行所有的`before_enqueue hook`,并且校验所有`before_enqueue hook`执行结果,如果有为`false`,则直接`return`,否则创建`job`,并且执行所有的`after_enqueue hook`.

我们再仔细看看 [Job.create](https://github.com/resque/resque/blob/master/lib/resque/job.rb#L40).

```ruby
# Creates a job by placing it on a queue. Expects a string queue
# name, a string class name, and an optional array of arguments to
# pass to the class' `perform` method.
#
# Raises an exception if no queue or class is given.
def self.create(queue, klass, *args)
  coder = Resque.coderhttps://github.com/resque/resque/blob/master/lib/resque/json_coder.rb
  Resque.validate(klass, queue)

  if Resque.inline?
    # Instantiating a Resque::Job and calling perform on it so callbacks run
    # decode(encode(args)) to ensure that args are normalized in the same
    # manner as a non-inline job
    payload = {'class' => klass, 'args' => coder.decode(coder.encode(args))}

    new(:inline, payload).perform
  else
    Resque.push(queue, 'class' => klass.to_s, 'args' => args)
  end
end
```
关键的地方来了!!!    
额,先稍等,一步一步来.

##### coder

```ruby
#https://github.com/resque/resque/blob/master/lib/resque.rb#L84
# Encapsulation of encode/decode. Overwrite this to use it across Resque.
# This defaults to JSON for backwards compatibility.
def coder
  @coder ||= JsonCoder.new
end
```
不难看出,是用来解析队列参数`args`的,这里默认是 [JsonCoder](https://github.com/resque/resque/blob/master/lib/resque/json_coder.rb),
用`JSON`来解析参数.

##### Resque.validate

前文已经讲过,用来校验队列名以及`job class`.

##### Resque.push

如果非`inline`,将`job`压入队列,等待异步执行.

##### perform

如果`inline`,直接执行`job`.可以具体看看是怎么执行的.    
先获取`job`参数,再用参数`new`一个`job`出来,然后执行`perform`方法.关键就在`perform`方法,这个方法就是去真正执行`job`了.    
下面再来具体看看到底是怎么`perform`的.    

```ruby
# https://github.com/resque/resque/blob/master/lib/resque/job.rb#L134
# Attempts to perform the work represented by this job instance.
# Calls #perform on the class given in the payload with the
# arguments given in the payload.
def perform
  begin
    hooks = {
      :before => before_hooks,
      :around => around_hooks,
      :after => after_hooks
    }
    JobPerformer.new.perform(payload_class, args, hooks)
    # If an exception occurs during the job execution, look for an
    # on_failure hook then re-raise.
  rescue Object => e
    run_failure_hooks(e)
    raise e
  end
end

#  https://github.com/resque/resque/blob/master/lib/resque/job_performer.rb#L12
# This is the actual performer for a single unit of work.  It's called
# by Resque::Job#perform
# Args:
#   palyoad_class: The class to call ::perform on
#   args: An array of args to pass to the payload_class::perform
#   hook: A hash with keys :before, :after and :around, all arrays of
#         methods to call on the payload class with args
def perform(payload_class, args, hooks)
  @job      = payload_class
  @job_args = args || []
  @hooks    = hooks

  # before_hooks can raise a Resque::DontPerform exception
  # in which case we exit this method, returning false (because
  # the job was never performed)
  return false unless call_before_hooks
  execute_job
  call_hooks(:after)

  performed?
end

# https://github.com/resque/resque/blob/master/lib/resque/job_performer.rb#L28
def call_before_hooks
  begin
    call_hooks(:before)
    true
  rescue Resque::DontPerform
    false
  end
end

# https://github.com/resque/resque/blob/master/lib/resque/job_performer.rb#L37
def execute_job
  # Execute the job. Do it in an around_perform hook if available.
  if hooks[:around].empty?
    perform_job
  else
    call_around_hooks
  end
end

# https://github.com/resque/resque/blob/master/lib/resque/job_performer.rb#L67
def perform_job
  result = job.perform(*job_args)
  job_performed
  result
end
```

由上述代码可以看出,其实真正执行的方法还是`job class`的`perform`方法,并且会去执行诸如`before_perform` `around_perform` `after_perform`等为前缀的`job class`方法.
整个`Resque`同步执行的过程就是这样!

### 但是

看完代码之后,我直接提了个[issue](https://github.com/resque/resque/issues/1026),
不知道是不是没开发完还是怎么回事!`inline`的描述和真实的行为竟然不一样!!!`inline`为`true`,还是会去执行`callbacks`!

### 最后

看了这篇文章后,脑子里应该能有一个大概的流程图,知道`Resque`大概的执行过程,这样我的目标就算达到了.后面我会再写一篇*Resque 异步执行*的文章,其实都是一样的,这个留到下次文章再分析.
