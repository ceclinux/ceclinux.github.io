[Sinatra](https://github.com/sinatra/sinatra)是一个极简的Web框架，目前代码仅仅17000行，相比而言，`rails` 5.1光Ruby代码就超过了220000，完全是就是庞然大物。对于初学者我来说，17000行代码要是真的读完，还是比较虚。正好，Sinatra的主要作者之一，[rkh](https://github.com/rkh)重写了`Sinatra`，并实现了`Sinatra`的核心逻辑[almost-sinatra](https://github.com/rkh/almost-sinatra)，仅仅用了6行代码，你没看错，**真的只有6行**。这6行里面，作者在`Rack`里面动了很多手脚，用了很多元编程的技巧，值得学习。

当然，为了压缩代码，作者做了很多丧心病狂的事情，比如给变量命超短的名字，比如不停的用`;`来避免换行-_-，但是没关系，我已经把它重重构好加上注释了。原代码如下，重构后的代码我放在[这里](https://github.com/ceclinux/almost_sinatra_refactored)

```ruby
%w.rack tilt backports date INT TERM..map{|l|trap(l){$r.stop}rescue require l};$u=Date;$z=($u.new.year + 145).abs;puts "== Almost Sinatra/No Version has taken the stage on #$z for development with backup from Webrick"
$n=Sinatra=Module.new{extend Rack;a,D,S,q=Rack::Builder.new,Object.method(:define_method),/@@ *([^\n]+)\n(((?!@@)[^\n]*\n)*)/m
%w[get post put delete].map{|m|D.(m){|u,&b|a.map(u){run->(e){[200,{"Content-Type"=>"text/html"},[a.instance_eval(&b)]]}}}}
Tilt.default_mapping.lazy_map.map{|k,v|D.(k){|n,*o|$t||=(h=$u._jisx0301("hash, please");File.read(caller[0][/^[^:]+/]).scan(S){|a,b|h[a]=b};h);Kernel.const_get(v[0][0]).new(*o){n=="#{n}"?n:$t[n.to_s]}.render(a,o[0].try(:[],:locals)||{})}}
%w[set enable disable configure helpers use register].map{|m|D.(m){|*_,&b|b.try :[]}};END{Rack::Handler.get("webrick").run(a,Port:$z){|s|$r=s}}
%w[params session].map{|m|D.(m){q.send m}};a.use Rack::Session::Cookie;a.use Rack::Lock;D.(:before){|&b|a.use Rack::Config,&b};before{|e|q=Rack::Request.new e;q.params.dup.map{|k,v|params[k.to_sym]=v}}}
```

好的，虽然重构了一下，这些代码的信息量依然巨大，我们一段一段的来看

```ruby
%w.rack tilt backports date INT TERM..map do |l|
    trap(l){$r.stop}rescue require l
end
```
一开始作者就拼命的想要省代码数，这里作者做了两件事情

- 引入`rack tilt backports date`等库 
- 当程序接收到`TERM`或者`INT`时，调用`$r.stop`

那它们是怎么被区分开来的呢？你可以把`rescue require l`去掉，运行后就会报错

```
almost_sinatra.rb:3:in `trap': unsupported signal SIGrack (ArgumentError
```
所以报错的，都被`require`了


至于`$r`是什么东西？，后面会提到。


```ruby
$curr_date=Date
$port_num=($curr_date.new.year + 145).abs
puts "== Almost Sinatra/No Version has taken the stage on #$port_num for development with backup from Webrick"
```
这三行非常简单，它打印了一行说明，并设置端口号为当前年 + 145

```ruby
$n=Sinatra=Module.new{
    extend Rack
}
```
继续向下看，有一个很大的代码块。这里新建了一个匿名Module并赋值给了`Sinatra`和`$n`，在这个Module中，这里`extend Rack`即继承Rack中所有的方法到当前对象中，由于当前对象`self`就是这个Module，所有相当于继承了所有Rack的实例方法到当前所建立的Module的类方法中，**然并软**，`Rack`模块偏偏没有实例方法，所有去掉这一行也能运行。

```ruby
    rack_builder = Rack::Builder.new
    method_obj = Object.method(:define_method)
    template_regex = /@@ *([^\n]+)\n(((?!@@)[^\n]*\n)*)/m
    rack_request = nil
```

这里面新建了四个变量，第一个`Rack::Builder.new`，是一个方便构建`rack`应用的工具，比如说这样，
```ruby
require 'rack/lobster'
app = Rack::Builder.new do
    use Rack::CommonLogger
    use Rack::ShowExceptions
    map "/lobster" do
      use Rack::Lint
      run Rack::Lobster.new
    end
  end

  run app
```

像写DSL一样！至于`Object.method(:define_method)`，是一个`Method`对象表示的方法，可以在以后使用`Method#call`方法对它进行调用（这里为了省代码，省略写成`.()`），也就是是说，我们把`define_method`这个方法拉出来了（后面我们会反复用到它）。`rack_request`暂定为空，`template_regex`这个多行`regex`接下来也会谈到。


```ruby
 %w[get post put delete].map do |m|
        method_obj.(m) do |u,&b|
            rack_builder.map(u){             
                run lambda { |env|
                    [
                        200,
                        {"Content-Type"=>"text/html"},
                        #instance_eval在一个对象的向下文中执行块
                        [rack_builder.instance_eval(&b)]
                    ]
                }
            }
        end
    end
```
好，我们开始使用`method_obj`(`Object.method(:define_method)`),它的使用姿势是`Method#call`，可以简写成`.()`，这里`method_obj.(m)`相当于构建了四个方法，分别是`get post put delete`，在每个方法中，我利用它传进来的参数和`Rack::Builder#map`方法建立路由，比如说，`get`方法就是

```ruby
def get(u, &b)
    map u do
        run lambda { |env|
                    [
                        200,
                        {"Content-Type"=>"text/html"},
                        #instance_eval在一个对象的向下文中执行块
                        [rack_builder.instance_eval(&b)]
                    ]
                }
    end
end
```
看看前面`Rack::Builder`的介绍，是不是很熟悉？至于为何写成了这样，你就需要懂`rack`，具体`rack`介绍你可以看[这里](https://ruby-china.org/topics/31592)也可以看源代码。

继续往下看

```ruby
 Tilt.default_mapping.lazy_map.map do |k,v|
        method_obj.(k){
            |n,*o| 

            $t||=(
                # https://ruby-doc.org/stdlib-2.3.1/libdoc/date/rdoc/Date.html#method-c-_jisx0301
                # Returns a hash of parsed elements.
                h=$curr_date._jisx0301("hash, please")
                File.read(caller[0][/^[^:]+/]).scan(template_regex){|a,b|h[a]=b}
                h
            )
            # n=="#{n}" to test n is a string
            Kernel.const_get(v[0][0]).new(*o){n=="#{n}"?n:$t[n.to_s]}
            .render(rack_builder,o[0]
            .try(:[],:locals)||{})
        }
    end
```
这里用到了一个叫`Tilt`库，`Tilt`帮你绑定了不同的模版引擎并封装好，以让人以统一的方式使用它。 简单说，这个方法遍历了所有的`Tilt.default_mapping.lazy_map`，它的每个`v`长成这样，

```
 [[["Tilt::ErubiTemplate", "tilt/erubi"],
  ["Tilt::ErubisTemplate", "tilt/erubis"],
  ["Tilt::ERBTemplate", "tilt/erb"]],
 [["Tilt::ErubiTemplate", "tilt/erubi"],
  ["Tilt::ErubisTemplate", "tilt/erubis"],
  ["Tilt::ERBTemplate", "tilt/erb"]],
 [["Tilt::ErubisTemplate", "tilt/erubis"]],
 [["Tilt::ErubiTemplate", "tilt/erubi"]],
 [["Tilt::PandocTemplate", "tilt/pandoc"],
```

他把所有模版引擎的名字作为定义的函数的名字，比如说`haml`。在这个函数里面，他先读取了我们写的模版文件，把其中的信息保存给了`t`（对，这个时候`template_regex`起了作用，他把整个文件切成了`String`数组），总之，如果存在这样一个模板文件，

```
@@ index
%html
  %head
    %title= @title
  %body
    %a{:href => '/hello?name=World'} Say hello!
    %a{:href => '/counter'} Show Counter

@@ hello
Hello <%= name %>!
```

最后`$t`变成了一个哈希，长成了这样，所以当你调用`haml :index`的时候，实际上`:index`作为键值帮你找到了相应的模板`$t[n.to_s]`，然后就被渲染出来啦。

```
{"index"=>
  "%html\n  %head\n    %title= @title\n  %body\n    %a{:href => '/hello?name=World'} Say hello!\n    %a{:href => '/counter'} Show Counter\n\n",
 "hello"=>"Hello <%= name %>!\n"}
```

然后使用`Kernel.const_get(v[0][0])`（对于`haml`实际上就是`Tilt::HamlTemplate`），然后`render`他，你当然可以传在`render`的时候传进`locals`变量（利用`try(:[],:locals)||{})`都把它引进来了）。

你会觉得很奇怪，`try`是哪里来的，这又不是`rails`? 其实，这是`backports`从`rails`里面偷来的[https://github.com/marcandre/backports/blob/5c876824f52ace6733ee00ae188f4a7e0a12ef32/lib/backports/rails/kernel.rb](https://github.com/marcandre/backports/blob/5c876824f52ace6733ee00ae188f4a7e0a12ef32/lib/backports/rails/kernel.rb)。

你依然会觉得奇怪，`h=$curr_date._jisx0301("hash, please")`是什么意思？其实这是作者的恶趣味= =|，`Date._jisx0301`只有传入符合格式的数组才会返回一个非空哈希，像这样

```ruby
Date._jisx0301("H13.02.03") # Date._jisx0301("H13.02.03")
```
不然就是一个空哈希而已

```ruby
%w[set enable disable configure helpers use register].each do |m|
        method_obj.(m) do |*_,&b| 
            b.try :[]
        end
    end
```
读到这里，你又发现作者开了个玩笑，他定义了一堆方法，然后当调用这方法时，只是运行方法传入的block而已，也就是说，
`enable :session`和`disable :session`其实都一样……


```ruby
 END{Rack::Handler.get("webrick").run(rack_builder,Port:$port_num){|s|$r=s}}
```
`END`表示该代码块总是最后运行，这里调用了`Rack::Handler::WEBrick#run`方法，并传入了代码块，然后把`WEBrick`服务器赋值给前面提到的`$r`，这样，`$r.stop`就可以关闭服务器啦。

```ruby
@server = ::WEBrick::HTTPServer.new(options)
yield @server  if block_given?
@server.start
```

```ruby
%w[params session].map{|m|method_obj.(m){rack_request.send m}};
```
这一行，所有`params`和`session`的调用都被**转发给了**`Rack::Request#params`和`Rack::Request#session`

```ruby
 rack_builder.use Rack::Session::Cookie
 rack_builder.use Rack::Lock
```

这里`rack_builder`使用了两个中间件，其中`Rack::Lock`使用`mutex`控制了每个请求，保证了每个请求是**同步的**。而`Rack::Session::Cookie`保证服务器客户端上简单的`cookie-session`控制。

```ruby
method_obj.(:before){|&b|rack_builder.use Rack::Config,&b}
before do |e|
    rack_request=Rack::Request.new e
    rack_request.params.dup.map{|k,v|params[k.to_sym]=v}
end
```
最后这几行代码中，先是定义了`before`方法，并把传入的块当作了`rack_builder.use`的参数。然后调用了`before`，将`rack_request`中`params`的`key`都转化成`symbol`。至于`rack_builder.use`和`Rack::Config`是什么鬼，代码都很简单，看一眼便知
```ruby
    def use(middleware, *args, &block)
      if @map
        mapping, @map = @map, nil
        @use << proc { |app| generate_map app, mapping }
      end
      @use << proc { |app| middleware.new(app, *args, &block) }
    end

    class Config
    def initialize(app, &block)
      @app = app
      @block = block
    end

    def call(env)
      @block.call(env)
      @app.call(env)
    end
  end
```

你现在看看`Sinatra`应用的例子[https://github.com/rkh/almost-sinatra/blob/master/example.rb](https://github.com/rkh/almost-sinatra/blob/master/example.rb)，是不是像开启写轮眼一样，里面的封装黑盒已经一清二楚了呢？^_^
