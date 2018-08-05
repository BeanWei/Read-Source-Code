# Flask源码解析

########################################
# flask简介                            #
########################################
Flask: 微框架，源码少，扩展性强。

核心依赖: werkzeug、jinjia。
werkzeug：负责逻辑模块；jinjia: 负责模板渲染

Q: flask 直接使用 jinja2 而不是把这部分也做成可扩展的看起来有悖它的设计原则?
A: flask 是个写网页的 web 框架，不像 flask-restful 可以专门做 json/xml 数据接口，必须提供模板功能，不然用户就无法使用。而如果不绑定一个模板库的话，有三种方法：自己写一个模板引擎、封装一个可扩展的模板层，用户可以自己选择具体的模板引擎、或者让用户自己处理模板。但是这些方法要么增加实现的复杂度，要么增加了使用的复杂度。

### werkzeug
定位: http和wsgi相关的工具集，为web框架提供帮助函数。


########################################
# 应用启动流程                          #
########################################

> 所有的 python web 框架都要遵循 WSGI 协议。WSGI 中有一个非常重要的概念，每个 python web 应用都是一个可调用( callable )的对象。
![img](https://uploads.toptal.io/blog/image/91961/toptal-blog-image-1452784558794-7851992813e17ce0d5ca9802cf7ac719.jpg)
### flask中的 wsgi
在flask中，这个对象就是 app = Flask(__name__) 创建出来的 app, 要运行 web 应用就必须有 web server, 由 werzeug 提供 WSGISerevr。

flask最简实例:
```
from flask import Flak
app = Flask(__name__)
@app.route('/')
def hello():
    return 'hello'
if __name__ == '__main__':
    app.run()
```
app = Flask(__name__) 就是Application部分。

那么 Server 部分在哪呢？隐藏在了 app.run() 里面

## flask run 方法源码
```
# 只看主要部分
def run(self, host=None, port=None, debug=None, **options):
    if debug is not None:
        self.debug = bool(debug)
    #内部默认启动地址和端口
    _host = '127.0.0.1'
    _port = 5000
    server_name = self.config.get('SERVER_NAME')
    sn_host, sn_port = None, None

    if server_name:
        sn_host, _, sn_port = server_name.partition(':)

    host = host or sn_host or _host
    port = int(port or sn_port or _port) 
    #调用 werkzeug 
    from werkzug.serving import run_simple
    try:
        # slef 参数就是要执行的 web application
        run_simple(host, port, self, **options)
    finally:
        self._got_first_request = False   
```
流程：WSGIServer 监听到指定的端口，将收到的http请求解析为WSGI格式然后调用 app 去执行处理的逻辑。
执行逻辑代码（werkzeug.serving:WSGIRequestHandler 的 run_wsgi 中）：
```
def execute(app):
    application_iter = app(environ, start_response)
    try:
        for data in application_iter:
            write(data)
        if not headers_set:
            write(b'')
    finally:
        if hasattr(application_iter, 'close'):
            application_iter.close()
            application_iter = None
```
可以看到 application_iter = app(environ, start_response) 就是调用代码获取结果的地方。

要调用 app 实例，那么它就需要定义了 __call__ 方法，我们找到 flask.app：Flask 对应的内容：
```
def wsgi_app(self, environ, start_response):
    #创建请求上下文
    ctx = self.request_context(environ)
    error = None
    try:
        try:
            #入栈
            ctx.push()
            #分发请求，根据请求路径通过路由找到对应的处理函数
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        except:
            error = sys.exc_info()[1]
            raise
        return response(environ, start_response)
    finally:
        #请求出栈(不管请求结果如何)
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)

def __call__(self, environ, start_response):
    return self.wsgi_app(environ, start_response)  
```

继续看 full_dsipatch_request 的代码
```
def full_dispatch_request(self):
    self.try_trigger_before_first_request_functions()
    try:
        request_started.send(self)
        rv = self.preprocess_request()
        if rv is None:
            #请求的hooks处理:http://flask.pocoo.org/docs/0.12/reqcontext/#callbacks-and-errors
            rv = self.dispatch_request()
    except Exception as e:
        rv = self.handle_user_exception(e)
    #self.finalize_request: 将结果转换成Response对象
    return self.finalize_request(rv)
```
dispatch_request 要做的就是找到我们的处理函数，并返回调用的结果，也就是路由的过程。


########################################
# 路由                                 #
########################################
一个 web 应用不同的路径会有不同的处理函数，路由就是根据请求的 URL 找到对应处理函数的过程。

## flask 构建路由的两种方法
1. 通过 @app.route() decorator。
2. 通过 app.add_url_rule, 这个方法的签名为 add_url_rule(self, rule, endpoint=None, view_func=None, **options)，参数的含义如下：
    1. rule： url 规则字符串，可以是静态的 /path，也可以包含 /
    2. endpoint：要注册规则的 endpoint，默认是 view_func 的名字
    3. view_func：对应 url 的处理函数，也被称为视图函数
也就是说下面这两种写法等价：
```
@app.route('/')
def f():
    pass
```
也可以写成
```
def f():
    pass
app.add_url_rule('/', 'f', f)
```

其实，还有一种方法来构建路由规则——直接操作 app.url_map 这个数据结构。不过这种方法并不是很常用，因此就不展开了。

注册路由规则的时候，flask 内部做了哪些东西呢？我们来看看 route 方法：
```
def route(self, rule, **options):
    def decorator(f):
        endpoint = options.pop('endpoint', None)
        self.add_url_rule(rule, endpoint, f, **options)
        return f
    return decorator
```
route 方法内部也是调用了add_url_rule只不过包装成了装饰器

```
def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
    methods = options.pop('methods', None)

    rule = self.url_rule_class(rule, methods=methods, **options)
    self.url_map.add(rule)
    if view_func is not None:
        old_func = self.view_functions.get(endpoint)
        #需要注意的是：每个视图函数的 endpoint 必须是不同的，否则会报 AssertionError。
        if old_func is not None and old_func != view_func:
            raise AssertionError('View function mapping is overwriting an '
                                 'existing endpoint function: %s' % endpoint)
        self.view_functions[endpoint] = view_func
```
上面这段代码省略了处理 endpoint 和构建 methods 的部分逻辑，可以看到它主要做的事情就是更新 self.url_map 和 self.view_functions 两个变量。

url_map 是 werkzeug.routeing:Map 类的对象，rule 是 werkzeug.routing:Rule 类的对象，view_functions 就是一个字典

## werkzeug 路由逻辑
```
>>> m = Map([
...     Rule('/', endpoint='index'),
...     Rule('/downloads/', endpoint='downloads/index'),
...     Rule('/downloads/<int:id>', endpoint='downloads/show')
... ])
>>> urls = m.bind("example.com", "/")
>>> urls.match("/", "GET")
('index', {})
>>> urls.match("/downloads/42")
('downloads/show', {'id': 42})

>>> urls.match("/downloads")
Traceback (most recent call last):
  ...
RequestRedirect: http://example.com/downloads/
>>> urls.match("/missing")
Traceback (most recent call last):
  ...
NotFound: 404 Not Found
```

上面的代码演示了 werkzeug 最核心的路由功能：添加路由规则（也可以使用 m.add），把路由表绑定到特定的环境（m.bind），匹配url（urls.match）。正常情况下返回对应的 endpoint 名字和参数字典，可能报重定向或者 404 异常。

可以发现，endpoint 在路由过程中非常重要。werkzeug 的路由过程，其实是 url 到 endpoint 的转换：通过 url 找到处理该 url 的 endpoint。至于 endpoint 和 view function 之间的匹配关系，werkzeug 是不管的，而上面也看到 flask 是把这个存放到字典中的。

我们回头看 flask 源码中的 dispatch_request，继续探寻路由匹配的逻辑：
```
def dispatch_request(self):
    req = _request_ctx_stack.top.request
    if req.routing_exception is not None:
        self.raise_routing_exception(req)
    rule = req.url_rule

    # dispatch to the handler for that endpoint
    return self.view_functions[rule.endpoint](**req.view_args)
```

这个方法做的事情就是找到请求对象 request，获取它的 endpoint，然后从 view_functions 找到对应 endpoint 的 view_func ，把请求参数传递过去，进行处理并返回。view_functions 中的内容，我们已经看到，是在构建路由规则的时候保存进去的；那请求中 req.url_rule 是什么保存进去的呢？它的格式又是什么？

我们可以先这样理解：_request_ctx_stack.top.request 保存着当前请求的信息，在每次请求过来的时候，flask 会把当前请求的信息保存进去，这样我们就能在整个请求处理过程中使用它。(至于怎么做到并发情况下信息不会相互干扰错乱，我们看下一部分。)

_request_ctx_stack 中保存的是 RequestContext 对象，它出现在 flask/ctx.py 文件中，和路由相关的逻辑如下：
```
class RequestContext(object):
	def __init__(self, app, environ, request=None):
        self.app = app
        self.request = request
        self.url_adapter = app.create_url_adapter(self.request)
        self.match_request()

    def match_request(self):
        """Can be overridden by a subclass to hook into the matching
        of the request.
        """
        try:
            url_rule, self.request.view_args = \
                self.url_adapter.match(return_rule=True)
            self.request.url_rule = url_rule
        except HTTPException as e:
            self.request.routing_exception = e


class Flask(_PackageBoundObject):
    def create_url_adapter(self, request):
        """Creates a URL adapter for the given request.  The URL adapter
        is created at a point where the request context is not yet set up
        so the request is passed explicitly.
        """
        if request is not None:
            return self.url_map.bind_to_environ(request.environ,
                server_name=self.config['SERVER_NAME'])
```

在初始化的时候，会调用 app.create_url_adapter 方法，把 app 的 url_map 绑定到 WSGI environ 变量上（bind_to_environ 和之前的 bind 方法作用相同）。最后会调用 match_request 方法，这个方式调用了 url_adapter.match 方法，进行实际的匹配工作，返回匹配的 url rule。而我们之前使用的 url_rule.endpoint 就是匹配的 endpoint 值。

整个 flask 的路由过程就结束了，总结一下大致的流程：
* 通过 @app.route 或者 app.add_url_rule 注册应用 url 对应的处理函数
* 每次请求过来的时候，会事先调用路由匹配的逻辑，把路由结果保存起来
* dispatch_request 根据保存的路由结果，调用对应的视图函数

## match 实现
了解完 flask 的路由流程，但是还没有说到最核心的问题：werkzeug 中是怎么实现 match 方法的。Map 保存了 Rule 列表，match 的时候会依次调用其中的 rule.match 方法，如果匹配就找到了 match。Rule.match 方法的代码如下：
```
def match(self, path):
    if not self.build_only:
        m = self._regex.search(path)
        if m is not None:
            groups = m.groupdict()

            result = {}
            for name, value in iteritems(groups):
                try:
                    value = self._converters[name].to_python(value)
                except ValidationError:
                    return
                result[str(name)] = value
            if self.defaults:
                result.update(self.defaults)

            return result
```

它的逻辑是这样的：用实现 compile 的正则表达式去匹配给出的真实路径信息，把所有的匹配组件转换成对应的值，保存在字典中（这就是传递给视图函数的参数列表）并返回。