> 本文是由[Inline caching in MRI](http://tenderlovemaking.com/2015/12/23/inline-caching-in-mri.html) 翻译而来，作者[@tenderlove](https://twitter.com/tenderlove)，他是[ruby的代码贡献者之一](https://github.com/ruby/ruby/graphs/contributors)，我已得到他本人的口头授权。

# MRI中的inline缓存

每一次你调用函数，Ruby会试图找到那个函数，然后再调用它。这个过程可能对性能耗费巨大。所以那些写虚拟机的程序员就会想方设法把函数查找这个过程缓存起来，以便于加快查找速度。MRI（其它大多数Ruby虚拟机也一样）使用的是inline缓存。我们都知道什么是缓存，那么inline缓存是什么鬼？

## 什么是inline缓存

Inline缓存是伴随Ruby字节码产生的“行内”缓存。比如以下程序。

```ruby
def foo bar, baz
  bar.baz
  baz.baz
end

ins = RubyVM::InstructionSequence.of method(:foo)
puts ins.disasm
```

如何你运行以上程序，你会看到和下面差不多的内容

```
$ ruby x.rb 
== disasm: #<ISeq:foo@x.rb>=============================================
local table (size: 3, argc: 2 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 3] bar<Arg>   [ 2] baz<Arg>   
0000 trace            8                                               (   1)
0002 trace            1                                               (   2)
0004 getlocal_OP__WC__0 3
0006 opt_send_without_block <callinfo!mid:baz, argc:0, ARGS_SIMPLE>, <callcache>
0009 pop              
0010 trace            1                                               (   3)
0012 getlocal_OP__WC__0 2
0014 opt_send_without_block <callinfo!mid:baz, argc:0, ARGS_SIMPLE>, <callcache>
0017 trace            16                                              (   4)
0019 leave            
```

在第`006`行和第`0014`行中，你可以看到三个差不多的结构，其中`opt_send_without_block`是虚拟机的命令，另外两个是这个命令的参数。你可以猜到，第一个结构由函数名和函数的参数等组成，而第二个结构，`<callcache>`则是我们所说的Inline缓存。

## 所以，inline缓存的键和值是什么？

答案是每个ruby虚拟机的对此实现不尽相同。以MRI为例，缓存有两个键，风别是“全局函数状态(global_method_state)”和“类序列号(class_serial)”。**缓存的值就是我们想要调用的方法**。

### 全局函数状态
全局函数状态就是一个**全局的**序列号，每次某些类被改变时，这个序列号就+1（比如说monkey patching）。你可以通过`RubyVM.stat`看到全局函数状态的值，比如说

```ruby
p RubyVM.stat

module Kernel
  def aaron
  end
end

p RubyVM.stat
```
如果你运行以上代码，你可以看到类似的结果。

```
{:global_method_state=>132, :global_constant_state=>824, :class_serial=>5664}
{:global_method_state=>133, :global_constant_state=>824, :class_serial=>5664}
```

给Kernel加一个函数会使全局函数状态数值改变。这并不是一个好的做法，因为它把所有的inline缓存都清空了。

# 类序列号(class_serial)

 另一种inline缓存键叫做类序列号(class serial number)。每个Ruby类都有一个序列号。我们可以再次调用`RubyVM.stat`来查看类序列号的变化

 ```ruby
 p RubyVM.stat

class A
end

p RubyVM.stat

class B
end

p RubyVM.stat
```

如果你运行以上代码，你会得到如下结果：

```
{:global_method_state=>132, :global_constant_state=>824, :class_serial=>5664}
{:global_method_state=>132, :global_constant_state=>825, :class_serial=>5666}
{:global_method_state=>132, :global_constant_state=>826, :class_serial=>5668}
```
你会发现类序列号一直在增长。其实，每个类都被赋予了一个序列号。只要你对类做了一些事。就会增大序列号。比如说，打开这个类。

```ruby
p RubyVM.stat

class A
  def foo; end
end

p RubyVM.stat

class A
  def bar; end
end

p RubyVM.stat
```
如果你运行以上代码，你会得到如下结果：
```
{:global_method_state=>132, :global_constant_state=>824, :class_serial=>5664}
{:global_method_state=>132, :global_constant_state=>825, :class_serial=>5667}
{:global_method_state=>132, :global_constant_state=>825, :class_serial=>5668}
```

我们没有办法检查每个类的序列号，但是通过`RubyVM.stat`可以使我们知道打开类所带来的副作用。所以**不要随意打开一个类**，因为他会使以这个类序列号为键的所有缓存都无效。

## 复习

如果我们有这段代码

```ruby
class A
  def baz; end
end

def foo bar
  bar.baz
end

foo(A.new)
```

那么，在`bar.baz`这一行便存在inline缓存。这个缓存的键为“全局函数状态“和`A`的类序列号。缓存的值就是`baz`函数在`A`定义的地方。

## 函数调用的类型与缓存

目前为止，我们已经知道inline缓存在那里，它的键与value是什么。但是我们还没讨论缓存的种类与大小等种种问题。**MRI中的cache大小是一**。你没看错，就是一。它只缓存了一个值。这被称作[“单态缓存”](https://en.wikipedia.org/wiki/Inline_caching#Monomorphic_inline_caching)，我们来看看这一个例子：
```ruby
class A
  def baz; end
end

def foo bar
  bar.baz
end

foo(A.new)
foo(A.new)
```
当`foo`第一次被调用的时候，MRI把目前的全局函数状态与缓存一开始的全局缓存状态作比较，同时，把A的类序列号和缓存的类序列号作比较。你可以在[这里](https://github.com/ruby/ruby/blob/7f71cdcf65ebd9354ceb77d46a0ff2a187e6a863/vm_insnhelper.c#L1126)看到源代码。因为当前没有缓存在，就被当作是缓存没有命中，所以Ruby就用本办法，从一步一步的找父类直到找到该函数，并缓存之。你也可以在[MRI源代码](https://github.com/ruby/ruby/blob/7f71cdcf65ebd9354ceb77d46a0ff2a187e6a863/vm_insnhelper.c#L1132-L1138)找到缓存值被保存。当这个方法被调用第二次的时候，缓存命中，我们不用在沿着路径慢慢找了。

我们来看看这一个例子：
```ruby
class A
  def baz; end
end

class B
  def baz; end
end

def foo bar
  bar.baz
end

loop do
  foo(A.new)
  foo(B.new)
end
```

在这个例子里面，我们**永远**都不会命中inline缓存。我们在调用函数`def foo bar`中，`bar`的类型一直在变，而缓存的键一部分就是类序列号。我们可以把这种函数调用叫做**“多态的”**函数调用因为`bar`的类型一直在变。

还有一个例子
```ruby
def foo bar
  bar.baz
end

loop do
  klass = Class.new {
    def baz
    end
  }
  foo(klass.new)
end
```
在这个例子中，缓存也**永远不会命中**。因为在这个例子中`bar`可以是**无限种类型**。我们把这种情况叫做**变态**。

# 复习

好了，现在我们有三种函数调用。单态的函数调用在函数调用的时候只会看到一种类型，多态的函数调用在函数调用的时候会看到多种类型。至于变态，可以算是多态的一种，因为类型太多了，我们很难用缓存来优化它。

Ruby的inline缓存只保存一个值，而且在单态函数调用的时候效率最高。而且，MRI的inline缓存只缓存最后遇到的值。这说明如果函数调用是多态的，但是类型变化的不是很快，那么即使会出现类型切换，函数查看的时间也会被平摊。

# 检查缓存命中情况
为了检查cache的恶名中情况，我魔改了下MRI，放在[这里](https://github.com/tenderlove/ruby/tree/IC_trace)。它可以帮助你检查inline缓存的命中情况。**请注意，它会降低Ruby的性能，请不要在生产环境中使用**。

我们先来看第一个例子，观察下他的缓存命中情况。

```ruby
class A
  def baz; end
end

def foo bar
  bar.baz
end

tp = TracePoint.new(:inline_cache_hit, :inline_cache_miss) do |x|
  p x => x.callcache_id
end

tp.enable
foo(A.new)
foo(A.new)
```
如果你运行以上代码，你会得到如下结果：
```
{#<TracePoint:inline_cache_miss@x.rb:14>=>[3386, 5666, "Class"]}
{#<TracePoint:inline_cache_miss@x.rb:14>=>[3387, 4052, "Object"]}
{#<TracePoint:inline_cache_miss@x.rb:6>=>[3380, 5667, "A"]}
{#<TracePoint:inline_cache_miss@x.rb:15>=>[3388, 5666, "Class"]}
{#<TracePoint:inline_cache_miss@x.rb:15>=>[3389, 4052, "Object"]}
{#<TracePoint:inline_cache_hit@x.rb:6>=>[3380, 5667, "A"]}
```

你会看到右边部分是一个由三部分组成的数组：1. 一个独有的函数调用ID 2. 类的序列号 3.类名。你可以看到最后一行是`A`的缓存命中（和我们期望中的一样）

还有一个例子：
```ruby
class A
  def baz; end
end

class B
  def baz; end
end

def foo bar
  bar.baz
end

tp = TracePoint.new(:inline_cache_hit, :inline_cache_miss) do |x|
  p x => x.callcache_id
end

tp.enable
2.times do
  foo(A.new)
  foo(B.new)
end
```


如果你运行以上代码，你会得到如下结果：
```
{#<TracePoint:inline_cache_miss@x.rb:18>=>[3391, 4869, "Fixnum"]}
{#<TracePoint:inline_cache_miss@x.rb:19>=>[3384, 5666, "Class"]}
{#<TracePoint:inline_cache_miss@x.rb:19>=>[3385, 4052, "Object"]}
{#<TracePoint:inline_cache_miss@x.rb:10>=>[3381, 5667, "A"]}
{#<TracePoint:inline_cache_miss@x.rb:20>=>[3386, 5669, "Class"]}
{#<TracePoint:inline_cache_miss@x.rb:20>=>[3387, 4052, "Object"]}
{#<TracePoint:inline_cache_miss@x.rb:10>=>[3381, 5670, "B"]}
{#<TracePoint:inline_cache_hit@x.rb:19>=>[3384, 5666, "Class"]}
{#<TracePoint:inline_cache_hit@x.rb:19>=>[3385, 4052, "Object"]}
{#<TracePoint:inline_cache_miss@x.rb:10>=>[3381, 5667, "A"]}
{#<TracePoint:inline_cache_hit@x.rb:20>=>[3386, 5669, "Class"]}
{#<TracePoint:inline_cache_hit@x.rb:20>=>[3387, 4052, "Object"]}
{#<TracePoint:inline_cache_miss@x.rb:10>=>[3381, 5670, "B"]}
```
你会看到一些缓存命中，但是没有一个是在`foo`函数里面的。

## 参考资料
- http://wiki.c2.com/?InlineCaching
- https://en.wikipedia.org/wiki/Inline_caching#Monomorphic_inline_caching
- https://bugs.ruby-lang.org/issues/11768
- https://books.google.com/books/about/Ruby_Under_a_Microscope.html?id=6AcvDwAAQBAJ
