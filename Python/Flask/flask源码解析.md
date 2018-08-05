# [Flask源码解析](http://cizixs.com/2017/01/10/flask-insight-introduction)

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


########################################
# 上下文                               #
########################################

> 每一段程序都有很多外部变量。只有像Add这种简单的函数才是没有外部变量的。一旦你的一段程序有了外部变量，这段程序就不完整，不能独立运行。你为了使他们运行，就要给所有的外部变量一个一个写一些值进去。这些值的集合就叫上下文。 – vzch

如果对 python 多线程比较熟悉的话，应该知道多线程中有个非常类似的概念 threading.local，可以实现多线程访问某个变量的时候只看到自己的数据。内部的原理说起来也很简单，这个对象有一个字典，保存了线程 id 对应的数据，读取该对象的时候，它动态地查询当前线程 id 对应的数据。flask python 上下文的实现也类似。

flask 中有两种上下文：application context 和 request context。上下文有关的内容定义在 globals.py 文件，文件的内容也非常短：
```
def _lookup_req_object(name):
    top = _request_ctx_stack.top
    if top is None:
        raise RuntimeError(_request_ctx_err_msg)
    return getattr(top, name)

def _lookup_app_object(name):
    top = _app_ctx_stack.top
    if top is None:
        raise RuntimeError(_app_ctx_err_msg)
    return getattr(top, name)

def _find_app():
    top = _app_ctx_stack.top
    if top is None:
        raise RuntimeError(_app_ctx_err_msg)
    return top.app

# context locals
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
current_app = LocalProxy(_find_app)
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))
g = LocalProxy(partial(_lookup_app_object, 'g'))
```

flask 提供两种上下文：application context 和 request context 。app lication context 又演化出来两个变量 current_app 和 g，而 request context 则演化出来 request 和 session。

这里的实现用到了两个东西：LocalStack 和 LocalProxy。它们两个的结果就是我们可以动态地获取两个上下文的内容，在并发程序中每个视图函数都会看到属于自己的上下文，而不会出现混乱。

LocalStack 和 LocalProxy 都是 werkzeug 提供的，定义在 local.py 文件中。在分析这两个类之前，我们先介绍这个文件另外一个基础的类 Local。Local 就是实现了类似 threading.local 的效果——多线程或者多协程情况下全局变量的隔离效果。下面是它的代码：
```
try:
    from greenlet import getcurrent as get_ident
except ImportError:
    try:
        from thread import get_ident
    except ImportError:
        from _thread import get_ident

class Local(object):
    __slots__ = ('__storage__', '__ident_func__')

    def __init__(self):
        # 数据保存在 __storage__ 中，后续访问都是对该属性的操作
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)

    def __call__(self, proxy):
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    # 清空当前线程/协程保存的所有数据
    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)

    # 下面三个方法实现了属性的访问、设置和删除。
    # 注意到，内部都调用 `self.__ident_func__` 获取当前线程或者协程的 id，然后再访问对应的内部字典。
    # 如果访问或者删除的属性不存在，会抛出 AttributeError。
    # 这样，外部用户看到的就是它在访问实例的属性，完全不知道字典或者多线程/协程切换的实现
    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
```

可以看到，Local 对象内部的数据都是保存在 __storage__ 属性的，这个属性变量是个嵌套的字典：map[ident]map[key]value。最外面字典 key 是线程或者协程的 identity，value 是另外一个字典，这个内部字典就是用户自定义的 key-value 键值对。用户访问实例的属性，就变成了访问内部的字典，外面字典的 key 是自动关联的。__ident_func 是 协程的 get_current 或者线程的 get_ident，从而获取当前代码所在线程或者协程的 id。

除了这些基本操作之外，Local 还实现了 __release_local__ ，用来清空（析构）当前线程或者协程的数据（状态）。__call__ 操作来创建一个 LocalProxy 对象。

LocalStack 是基于 Local 实现的栈结构。如果说 Local 提供了多线程或者多协程隔离的属性访问，那么 LocalStack 就提供了隔离的栈访问。下面是它的实现代码，可以看到它提供了 push、pop 和 top 方法。

__release_local__ 可以用来清空当前线程或者协程的栈数据，__call__ 方法返回当前线程或者协程栈顶元素的代理对象。

```
class LocalStack(object):
    """This class works similar to a :class:`Local` but keeps a stack
    of objects instead. """

    def __init__(self):
        self._local = Local()

    def __release_local__(self):
        self._local.__release_local__()

    def __call__(self):
        def _lookup():
            rv = self.top
            if rv is None:
                raise RuntimeError('object unbound')
            return rv
        return LocalProxy(_lookup)

    # push、pop 和 top 三个方法实现了栈的操作，
    # 可以看到栈的数据是保存在 self._local.stack 属性中的
    def push(self, obj):
        """Pushes a new item to the stack"""
        rv = getattr(self._local, 'stack', None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv

    def pop(self):
        """Removes the topmost item from the stack, will return the
        old value or `None` if the stack was already empty.
        """
        stack = getattr(self._local, 'stack', None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()

    @property
    def top(self):
        """The topmost item on the stack.  If the stack is empty,
        `None` is returned.
        """
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None
```

我们在之前看到了 request context 的定义，它就是一个 LocalStack 的实例：
```
_request_ctx_stack = LocalStack()
```

它会将当前线程或者协程的请求都保存在栈里，等使用的时候再从里面读取。
(Q: 为什么要用到栈结构，而不是直接使用 Local?)

LocalProxy 是一个 Local 对象的代理，负责把所有对自己的操作转发给内部的 Local 对象。LocalProxy 的构造函数介绍一个 callable 的参数，这个 callable 调用之后需要返回一个 Local 实例，后续所有的属性操作都会转发给 callable 返回的对象。

```
class LocalProxy(object):
    """Acts as a proxy for a werkzeug local.
    Forwards all operations to a proxied object. """
    __slots__ = ('__local', '__dict__', '__name__')

    def __init__(self, local, name=None):
        object.__setattr__(self, '_LocalProxy__local', local)
        object.__setattr__(self, '__name__', name)

    def _get_current_object(self):
        """Return the current object."""
        if not hasattr(self.__local, '__release_local__'):
            return self.__local()
        try:
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError('no object bound to %s' % self.__name__)

    @property
    def __dict__(self):
        try:
            return self._get_current_object().__dict__
        except RuntimeError:
            raise AttributeError('__dict__')

    def __getattr__(self, name):
        if name == '__members__':
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)

    def __setitem__(self, key, value):
        self._get_current_object()[key] = value
```

这里实现的关键是把通过参数传递进来的 Local 实例保存在 __local 属性中，并定义了 _get_current_object() 方法获取当前线程或者协程对应的对象。

NOTE：前面双下划线的属性，会保存到 _ClassName__variable 中。所以这里通过 “_LocalProxy__local” 设置的值，后面可以通过 self.__local 来获取。关于这个知识点，可以查看 [stackoverflow 的这个问题](http://stackoverflow.com/questions/1301346/the-meaning-of-a-single-and-a-double-underscore-before-an-object-name-in-python)。

然后 LocalProxy 重写了所有的魔术方法（名字前后有两个下划线的方法），具体操作都是转发给代理对象的。这里只给出了几个魔术方法，感兴趣的可以查看源码中所有的魔术方法

继续回到 request context 的实现：
```
_request_ctx_stack = LocalStack()
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))
```

_request_ctx_stack 是多线程或者协程隔离的栈结构，request 每次都会调用 _lookup_req_object 栈头部的数据来获取保存在里面的 requst context。

那么请求上下文信息是什么被放在 stack 中呢？还记得之前介绍的 wsgi_app() 方法有下面两行代码吗？
```
ctx = self.request_context(environ)
ctx.push()
```
每次在调用 app.__call__ 的时候，都会把对应的请求信息压栈，最后执行完请求的处理之后把它出栈。

我们来看看request_context， 这个 方法只有一行代码：
```
def request_context(self, environ):
    return RequestContext(self, environ)
```

它调用了 RequestContext，并把 self 和请求信息的字典 environ 当做参数传递进去。追踪到 RequestContext 定义的地方，它出现在 ctx.py 文件中，代码如下：

```
class RequestContext(object):
    """The request context contains all request relevant information.  It is
    created at the beginning of the request and pushed to the
    `_request_ctx_stack` and removed at the end of it.  It will create the
    URL adapter and request object for the WSGI environment provided.
    """

    def __init__(self, app, environ, request=None):
        self.app = app
        if request is None:
            request = app.request_class(environ)
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

    def push(self):
        """Binds the request context to the current context."""
        # Before we push the request context we have to ensure that there
        # is an application context.
        app_ctx = _app_ctx_stack.top
        if app_ctx is None or app_ctx.app != self.app:
            app_ctx = self.app.app_context()
            app_ctx.push()
            self._implicit_app_ctx_stack.append(app_ctx)
        else:
            self._implicit_app_ctx_stack.append(None)

        _request_ctx_stack.push(self)

        self.session = self.app.open_session(self.request)
        if self.session is None:
            self.session = self.app.make_null_session()

    def pop(self, exc=_sentinel):
        """Pops the request context and unbinds it by doing that.  This will
        also trigger the execution of functions registered by the
        :meth:`~flask.Flask.teardown_request` decorator.
        """
        app_ctx = self._implicit_app_ctx_stack.pop()

        try:
            clear_request = False
            if not self._implicit_app_ctx_stack:
                self.app.do_teardown_request(exc)

                request_close = getattr(self.request, 'close', None)
                if request_close is not None:
                    request_close()
                clear_request = True
        finally:
            rv = _request_ctx_stack.pop()

            # get rid of circular dependencies at the end of the request
            # so that we don't require the GC to be active.
            if clear_request:
                rv.request.environ['werkzeug.request'] = None

            # Get rid of the app as well if necessary.
            if app_ctx is not None:
                app_ctx.pop(exc)

    def auto_pop(self, exc):
        if self.request.environ.get('flask._preserve_context') or \
           (exc is not None and self.app.preserve_context_on_exception):
            self.preserved = True
            self._preserved_exc = exc
        else:
            self.pop(exc)

    def __enter__(self):
        self.push()
        return self

    def __exit__(self, exc_type, exc_value, tb):
        self.auto_pop(exc_value)
```

每个 request context 都保存了当前请求的信息，比如 request 对象和 app 对象。在初始化的最后，还调用了 match_request 实现了路由的匹配逻辑。

push 操作就是把该请求的 ApplicationContext（如果 _app_ctx_stack 栈顶不是当前请求所在 app ，需要创建新的 app context） 和 RequestContext 有关的信息保存到对应的栈上，压栈后还会保存 session 的信息； pop 则相反，把 request context 和 application context 出栈，做一些清理性的工作。

到这里，上下文的实现就比较清晰了：每次有请求过来的时候，flask 会先创建当前线程或者进程需要处理的两个重要上下文对象，把它们保存到隔离的栈里面，这样视图函数进行处理的时候就能直接从栈上获取这些信息。

NOTE：因为 app 实例只有一个，因此多个 request 共享了 application context。

到这里，关于 context 的实现和功能已经讲解得差不多了。还有两个疑惑没有解答。
1. 为什么要把 request context 和 application context 分开？每个请求不是都同时拥有这两个上下文信息吗？
2. 为什么 request context 和 application context 都有实现成栈的结构？每个请求难道会出现多个 request context 或者 application context 吗？
   
第一个答案是“灵活度”，第二个答案是“多 application”。虽然在实际运行中，每个请求对应一个 request context 和一个 application context，但是在测试或者 python shell 中运行的时候，用户可以单独创建 request context 或者 application context，这种灵活度方便用户的不同的使用场景；而且栈可以让 redirect 更容易实现，一个处理函数可以从栈中获取重定向路径的多个请求信息。application 设计成栈也是类似，测试的时候可以添加多个上下文，另外一个原因是 flask 可以多个 application 同时运行:
```
from werkzeug.wsgi import DispatcherMiddleware
from frontend_app import application as frontend
from backend_app import application as backend

application = DispatcherMiddleware(frontend, {
    '/backend':     backend
})
```
这个例子就是使用 werkzeug 的 DispatcherMiddleware 实现多个 app 的分发，这种情况下 _app_ctx_stack 栈里会出现两个 application context。

## 为什么要用 LocalProxy
为什么要使用 LocalProxy？不使用 LocalProxy 直接访问 LocalStack 的对象会有什么问题吗？

首先明确一点，Local 和 LocalStack 实现了不同线程/协程之间的数据隔离。在为什么用 LocalStack 而不是直接使用 Local 的时候，我们说过这是因为 flask 希望在测试或者开发的时候，允许多 app 、多 request 的情况。而 LocalProxy 也是因为这个才引入进来的！

我们拿 current_app = LocalProxy(_find_app) 来举例子。每次使用 current_app 的时候，他都会调用 _find_app 函数，然后对得到的变量进行操作。

如果直接使用 current_app = _find_app() 有什么区别呢？区别就在于，我们导入进来之后，current_app 就不会再变化了。如果有多 app 的情况，就会出现错误，比如：
```
from flask import current_app

app = create_app()
admin_app = create_admin_app()

def do_something():
    with app.app_context():
        work_on(current_app)
        with admin_app.app_context():
            work_on(current_app)
```

这里我们出现了嵌套的 app，每个 with 上下文都需要操作其对应的 app，如果不使用 LocalProxy 是做不到的。

对于 request 也是类似！但是这种情况真的很少发生，有必要费这么大的功夫增加这么多复杂度吗？

其实还有一个更大的问题，这个例子也可以看出来。比如我们知道 current_app 是动态的，因为它背后对应的栈会 push 和 pop 元素进去。那刚开始的时候，栈一定是空的，只有在 with app.app_context() 这句的时候，才把栈数据 push 进去。而如果不采用 LocalProxy 进行转发，那么在最上面导入 from flask import current_app 的时候，current_app 就是空的，因为这个时候还没有把数据 push 进去，后面调用的时候根本无法使用。

所以为什么需要 LocalProxy 呢？简单总结一句话：因为上下文保存的数据是保存在栈里的，并且会动态发生变化。如果不是动态地去访问，会造成数据访问异常。


########################################
# 请求                                 #
########################################

对于物理链路来说，请求只是不同电压信号，它根本不知道也不需要知道请求格式和内容到底是怎样的； 对于 TCP 层来说，请求就是传输的数据（二进制的数据流），它只要发送给对应的应用程序就行了； 对于 HTTP 层的服务器来说，请求必须是符合 HTTP 协议的内容； 对于 WSGI server 来说，请求又变成了文件流，它要读取其中的内容，把 HTTP 请求包含的各种信息保存到一个字典中，调用 WSGI app； 对于 flask app 来说，请求就是一个对象，当需要某些信息的时候，只需要读取该对象的属性或者方法就行了。

可以看到，虽然是同样的请求数据，在不同的阶段和不同组件看来，是完全不同的形式。因为每个组件都有它本身的目的和功能，这和生活中的事情一个道理：对于同样的事情，不同的人或者同一个人不同人生阶段的理解是不一样的。

我们知道要访问 flask 的请求对象非常简单，只需要 from flask import request：
```
from flask import request

with app.request_context(environ):
    assert request.method == 'POST'
```

它最后对应了 flask.wrappers:Request 类的对象。 这个类内部的实现虽然我们还不清楚，但是我们知道它接受 WSGI server 传递过来的 environ 字典变量，并提供了很多常用的属性和方法可以使用，比如请求的 method、path、args 等。 请求还有一个不那么明显的特性——它不能被应用修改，应用只能读取请求的数据。

这个类的定义很简单，它继承了 werkzeug.wrappers:Request，然后添加了一些属性，这些属性和 flask 的逻辑有关，比如 view_args、blueprint、json 处理等。它的代码如下：

```
from werkzeug.wrappers import Request as RequestBase

class Request(RequestBase):
    """
    The request object is a :class:`~werkzeug.wrappers.Request` subclass and
    provides all of the attributes Werkzeug defines plus a few Flask
    specific ones.
    """

    #: The internal URL rule that matched the request.  This can be
    #: useful to inspect which methods are allowed for the URL from
    #: a before/after handler (``request.url_rule.methods``) etc.
    url_rule = None

    #: A dict of view arguments that matched the request.  If an exception
    #: happened when matching, this will be ``None``.
    view_args = None

    @property
    def max_content_length(self):
        """Read-only view of the ``MAX_CONTENT_LENGTH`` config key."""
        ctx = _request_ctx_stack.top
        if ctx is not None:
            return ctx.app.config['MAX_CONTENT_LENGTH']

    @property
    def endpoint(self):
        """The endpoint that matched the request.  This in combination with
        :attr:`view_args` can be used to reconstruct the same or a
        modified URL.  If an exception happened when matching, this will
        be ``None``.
        """
        if self.url_rule is not None:
            return self.url_rule.endpoint

    @property
    def blueprint(self):
        """The name of the current blueprint"""
        if self.url_rule and '.' in self.url_rule.endpoint:
            return self.url_rule.endpoint.rsplit('.', 1)[0]

    @property
    def is_json(self):
        mt = self.mimetype
        if mt == 'application/json':
            return True
        if mt.startswith('application/') and mt.endswith('+json'):
            return True
        return False
```

这段代码没有什难理解的地方，唯一需要说明的就是 @property 装饰符能够把类的方法变成属性，这是 python 中经常见到的用法。

接着我们就要看 werkzeug.wrappers:Request：
```
class Request(BaseRequest, AcceptMixin, ETagRequestMixin,
              UserAgentMixin, AuthorizationMixin,
              CommonRequestDescriptorsMixin):

    """Full featured request object implementing the following mixins:

    - :class:`AcceptMixin` for accept header parsing
    - :class:`ETagRequestMixin` for etag and cache control handling
    - :class:`UserAgentMixin` for user agent introspection
    - :class:`AuthorizationMixin` for http auth handling
    - :class:`CommonRequestDescriptorsMixin` for common headers
    """
```

这个方法有一点比较特殊，它没有任何的 body。但是有多个基类，第一个是 BaseRequest，其他的都是各种 Mixin。 这里要讲一下 Mixin 机制，这是 python 多继承的一种方式，如果你希望某个类可以自行组合它的特性（比如这里的情况），或者希望某个特性用在多个类中，就可以使用 Mixin。 如果我们只需要能处理各种 Accept 头部的请求，可以这样做：
```
class Request(BaseRequest, AcceptMixin)
    pass
```

但是不要滥用 Mixin，在大多数情况下子类继承了父类，然后实现需要的逻辑就能满足需求。

我们先来看看 BaseRequest:
```
class BaseRequest(object):
    def __init__(self, environ, populate_request=True, shallow=False):
        self.environ = environ
        if populate_request and not shallow:
            self.environ['werkzeug.request'] = self
        self.shallow = shallow
```

能看到实例化需要的唯一变量是 environ，它只是简单地把变量保存下来，并没有做进一步的处理。Request 的内容很多，其中相当一部分是被 @cached_property 装饰的方法，比如下面这种：

```
@cached_property
def args(self):
    """The parsed URL parameters."""
    return url_decode(wsgi_get_bytes(self.environ.get('QUERY_STRING', '')),
                        self.url_charset, errors=self.encoding_errors,
                        cls=self.parameter_storage_class)

@cached_property
def stream(self):
    """The stream to read incoming data from.  Unlike :attr:`input_stream`
    this stream is properly guarded that you can't accidentally read past
    the length of the input.  Werkzeug will internally always refer to
    this stream to read data which makes it possible to wrap this
    object with a stream that does filtering.
    """
    _assert_not_shallow(self)
    return get_input_stream(self.environ)

@cached_property
def form(self):
    """The form parameters."""
    self._load_form_data()
    return self.form

@cached_property
def cookies(self):
    """Read only access to the retrieved cookie values as dictionary."""
    return parse_cookie(self.environ, self.charset,
                        self.encoding_errors,
                        cls=self.dict_storage_class)

@cached_property
def headers(self):
    """The headers from the WSGI environ as immutable
    :class:`~werkzeug.datastructures.EnvironHeaders`.
    """
    return EnvironHeaders(self.environ)
```

@cached_property 从名字就能看出来，它是 @property 的升级版，添加了缓存功能。我们知道 @property 能把某个方法转换成属性，每次访问属性的时候，它都会执行底层的方法作为结果返回。 @cached_property 也一样，区别是只有第一次访问的时候才会调用底层的方法，后续的方法会直接使用之前返回的值。 那么它是如何实现的呢？我们能在 werkzeug.utils 找到它的定义：

```
class cached_property(property):

    """A decorator that converts a function into a lazy property.  The
    function wrapped is called the first time to retrieve the result
    and then that calculated result is used the next time you access
    the value.

    The class has to have a `__dict__` in order for this property to
    work.
    """

    # implementation detail: A subclass of python's builtin property
    # decorator, we override __get__ to check for a cached value. If one
    # choses to invoke __get__ by hand the property will still work as
    # expected because the lookup logic is replicated in __get__ for
    # manual invocation.

    def __init__(self, func, name=None, doc=None):
        self.__name__ = name or func.__name__
        self.__module__ = func.__module__
        self.__doc__ = doc or func.__doc__
        self.func = func

    def __set__(self, obj, value):
        obj.__dict__[self.__name__] = value

    def __get__(self, obj, type=None):
        if obj is None:
            return self
        value = obj.__dict__.get(self.__name__, _missing)
        if value is _missing:
            value = self.func(obj)
            obj.__dict__[self.__name__] = value
        return value
```

这个装饰器同时也是实现了 __set__ 和 __get__ 方法的描述器。 访问它装饰的属性，就会调用 __get__ 方法，这个方法先在 obj.__dict__ 中寻找是否已经存在对应的值。如果存在，就直接返回；如果不存在，调用底层的函数 self.func，并把得到的值保存起来，再返回。这也是它能实现缓存的原因：因为它会把函数的值作为属性保存到对象中。

关于 Request 内部各种属性的实现，就不分析了，因为它们每个具体的实现都不太一样，也不复杂，无外乎对 environ 字典中某些字段做一些处理和计算。 接下来回过头来看看 Mixin，这里只用 AcceptMixin 作为例子：
```
class AcceptMixin(object):

    @cached_property
    def accept_mimetypes(self):
        return parse_accept_header(self.environ.get('HTTP_ACCEPT'), MIMEAccept)

    @cached_property
    def accept_charsets(self):
        return parse_accept_header(self.environ.get('HTTP_ACCEPT_CHARSET'),
                                   CharsetAccept)

    @cached_property
    def accept_encodings(self):
        return parse_accept_header(self.environ.get('HTTP_ACCEPT_ENCODING'))

    @cached_property
    def accept_languages(self):
        return parse_accept_header(self.environ.get('HTTP_ACCEPT_LANGUAGE'),
                                   LanguageAccept)
```

AcceptMixin 实现了请求内容协商的部分，比如请求接受的语言、编码格式、相应内容等。 它也是定义了很多 @cached_property 方法，虽然自己没有 __init__ 方法，但是也直接使用了 self.environ，因此它并不能直接使用，只能和 BaseRequest 一起出现。