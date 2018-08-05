# Flask源码解析

# flask简介
Flask: 微框架，源码少，扩展性强。

核心依赖: werkzeug、jinjia。
werkzeug：负责逻辑模块；jinjia: 负责模板渲染

Q: flask 直接使用 jinja2 而不是把这部分也做成可扩展的看起来有悖它的设计原则?
A: flask 是个写网页的 web 框架，不像 flask-restful 可以专门做 json/xml 数据接口，必须提供模板功能，不然用户就无法使用。而如果不绑定一个模板库的话，有三种方法：自己写一个模板引擎、封装一个可扩展的模板层，用户可以自己选择具体的模板引擎、或者让用户自己处理模板。但是这些方法要么增加实现的复杂度，要么增加了使用的复杂度。

### werkzeug
定位: http和wsgi相关的工具集，为web框架提供帮助函数。


# 应用启动流程
> 所有的 python web 框架都要遵循 WSGI 协议。WSGI 中有一个非常重要的概念，每个 python web 应用都是一个可调用( callable )的对象。
![img](https://uploads.toptal.io/blog/image/91961/toptal-blog-image-1452784558794-7851992813e17ce0d5ca9802cf7ac719.jpg)
### flask中的 wsgi
在flask中，这个对象就是 app = Flask(__name__) 创建出来的 app, 要运行 web 应用就必须有 web server, 由 werzeug 提供 WSGISerevr。

flask最简实例:
```
from flask import Flak
app = Flask(__name__)
@route('/')
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
