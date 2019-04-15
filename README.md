# TinyURL 项目

此项目来自于 Werkzeug 文档的 tutorial 部分([点我跳转](https://werkzeug.palletsprojects.com/en/0.15.x/tutorial/)) ，用于短链接的生成。项目使用到了 Jinja2 ， redis 以及 Werkzeug 等相关技术。

## 第一步：创建项目目录

开始之前，我们需要为应用创建如下的目录结构：

```
/shortly
  /static
  /templates
```

其中 `shortly` 并不是一个 Python 的 package (包)，它仅仅用来放置我们的文件。接下来的步骤我们会在这个目录下放置我们的主要 module (模块) 。 `static` 目录下的文件是应用的用户可以通过 HTTP 直接访问得到的，这也是我们放置 CSS 以及 JavaScript 文件的地方。 `templates` 目录是提供给 Jinja2 查找使用的。

## 第二步：基本架构

在 *shortly* 目录下创建 *shortly.py* 文件。首先我们需要导入一堆依赖。这里我们选择先将全部的依赖导入（尽管它们中的部分在这个步骤还没被使用，可能会让人觉得云里雾里）。

``` python
import os
import redis
import urlparse
from werkzeug.wrappers import Request, Response
from werkzeug.routing import Map, Rule
from werkzeug.exceptions import HTTPException, NotFound
from werkzeug.wsgi import SharedDataMiddleware
from werkzeug.utils import redirect
from jinja2 import Environment, FileSystemLoader
```

然后我们可以创建我们应用的基本架构，以及创建一个应用程序的实例。这里有个非必要的步骤：使用 WSGI 中间件来导出所有的 *static* 目录下的文件。

``` python
class Shortly(object):

    def __init__(self, config):
        self.redis = redis.Redis(config['redis_host'], config['redis_port'])

    def dispatch_request(self, request):
        return Response('Hello World!')

    def wsgi_app(self, environ, start_response):
        request = Request(environ)
        response = self.dispatch_request(request)
        return response(environ, start_response)

    def __call__(self, environ, start_response):
        return self.wsgi_app(environ, start_response)


def create_app(redis_host='localhost', redis_port=6379, with_static=True):
    app = Shortly({
        'redis_host': redis_host,
        'redis_port': redis_port
    })
    if with_static:
        app.wsgi_app = SharedDataMiddleware(app.wsgi_app, {
            '/static': os.path.join(os.path.dirname(__file__), 'static')
        })
    return app
```

现在我们可以加一小段代码，来启动一个本地的开发服务器(带有热更新和调试功能)：

``` python
if __name__ == '__main__':
    from werkzeug.serving import run_simple
    app = create_app()
    run_simple('127.0.0.1', 5000, app, use_debugger=True, use_reloader=True)
```

这里的基本思路是： `Shortly` 类是一个实际的 WSGI 应用程序。 `__call__` 方法实际上是直接调用了 `wsgi_app` 方法。这样做是为了让我们可以像在 `create_app` 函数中那样，对 `wsgi_app` 做一层包装来使用中间件。实际上的 `wsgi_app` 方法会创建一个 `Request` 对象，并且当需要返回一个 `Response` 对象时调用 `dispatch_request` 方法（这里的 `Response` 对象实际上是作为 WSGI 应用程序使用）。如你所见，我们创建的 `Shortly` 类以及 Werkzeug 中的任何请求对象都实现了 WSGI 的接口。因此，你甚至可以在 `dispatch_request` 方法中返回另一个 WSGI 对象。

`create_app` 工厂方法用于为我们的应用程序创建一个新的实例。它不仅仅可以为应用传入一些配置参数，还可以添加 WSGI 中间件来导出静态文件。

## 插曲：运行程序

现在你应该可以使用 Python 运行你的文件并看到在你本机的服务器信息：

``` shell
python shortly.py
 * Running on http；//127.0.0.1:5000
 * Restarting with reloader: stat() polling
```

## 第三步：环境

现在我们有了一个基本的应用程序类，我们可以让构造函数做一些有用的事情并在那里提供一些可以派上用场的 helper 。我们目前需要完成：模板渲染以及连接 redis 。所以让我们稍微扩展一下这个类：

``` python
def __init__(self, config):
    self.redis = redis.Redis(config['redis_host'], config['redis_port'])
    template_path = os.path.join(os.path.dirname(__file__), 'templates')
    self.jinja_env = Environment(
        loader=FileSystemLoader(template_path), autoescape=True)

def render_template(self, template_name, **context):
    t = self.jinja_env.get_template(template_name)
    return Response(t.render(context), mimetype='text/html')
```

## 第四步：路由

接下来是路由。**路由是将 URL 匹配和解析成我们可以使用的东西的过程**。 Werkzeug 提供了一个内置的灵活的路由系统，我们可以直接使用它。它的工作方式是：创建一个 [Map](https://werkzeug.palletsprojects.com/en/0.15.x/routing/#werkzeug.routing.Map) 实例并添加一堆 [Rule](https://werkzeug.palletsprojects.com/en/0.15.x/routing/#werkzeug.routing.Rule) 对象。每个 `rule` 都有一个模式(pattern)，它会尝试配对 URL 跟 *endpoint* (端点)。endpoint 通常是一个字符串，可用于 URL 的唯一标识。我们也可以用它来实现 URL reverse(URL 逆解析)，但这不是本 demo 要做的：

将其放到构造函数中：

``` python
self.url_map = Map([
    Rule('/', endpoint='new_url'),
    Rule('/<short_id>', endpoint='follow_short_link'),
    Rule('/<short_id>+', endpoint='short_link_details')
])
```

这里我们创建了一个 URL 的哈希表，里面包含三个规则 `rule` 。`/` 代表根 URL，它会调度一个函数来创建新的 URL 。下一个规则是找到对应 `short_id` 的短链接，而另一个有相似规则但是结尾处加了 `+` 的规则，表示显式对应短链接的详细信息。

那么我们如何找到从 endpoint 到具体函数的映射方式呢？那就随你的便了~在本 demo 中是通过调用类本身的 **`on_` + endpoint 名** 的函数来工作的：

``` python
# 更新的 dispatch_request 方法
def dispatch_request(self, request):
    adapter = self.url_map.bind_to_environ(request.environ)
    try:
        endpoint, values = adapter.match()
        return getattr(self, 'on_' + endpoint)(request, **values)
    except HTTPException, e:
        return e
```

我们将 URL 哈希表和当前的环境进行绑定，得到一个 **URLAdapter** 的返回。这个 adapter (适配器) 不仅可以用来匹配请求 request 还可以逆解析 URL 。这个 `match` 方法会返回匹对 URL 的 `endpoint` 的值以及一个 `values` 的字典。例如对于规则 `follow_short_link` 会得到一个变量值叫做 `short_id`。当我们访问 `http://localhost:5000/foo` 时我们会得到以下的值：

``` python
endpoint = 'follow_short_link'
values = {'short_id': u'foo'}
```

如果匹配不到任何东西，werkzeug 会抛出一个 [`NotFound` 异常](https://werkzeug.palletsprojects.com/en/0.15.x/exceptions/#werkzeug.exceptions.NotFound) ，它本质上是一个 [`HTTPException`](https://werkzeug.palletsprojects.com/en/0.15.x/exceptions/#werkzeug.exceptions.HTTPException) 。<u>所有的 HTTP 异常对 Werkzeug 来说都是 WSGI 应用程序，它会渲染一个默认的错误页(error page)</u>。因此我们只需要捕获它们并直接将这些错误本身返回出去。

如果程序工作良好，我们将调用 **`on_` + endpoint 名** 的函数并将 request 作为参数传递进去。其中所有的 URL 参数都将作为关键字参数。然后函数会返回一个 `Response` 对象以供应用程序返回。


## 第五步：第一个视图

这里是我们的第一个视图：创建新 URL 。

``` python
def on_new_url(self, request):
    error = None
    url = ''
    if request.method == 'POST':
        url = request.form['url']
        if not is_valid_url(url):
            error = 'Please enter a valid URL'
        else:
            short_id = self.insert_url(url)
            return redirect('/%s+' % short_id)
    return self.render_template('new_url.html', error=error, url=url)
```

这段逻辑非常容易理解：先检查是否为 POST 请求，然后检查 URL 是否合法，然后往数据库插入新的记录，最后重定向到详情页。这意味着我们需要写一个函数以及一个 helper 方法。下面对于验证 URL 已经足够了：

``` python
def is_valid_url(url):
    parts = urlparse.urlparse(url)
    return parts.scheme in ('http', 'https')
```

对于插入 URL 到数据库，我们需要以下方法：

``` python
def insert_url(self, url):
    short_id = self.redis.get('reverse-url:' + url)
    if short_id is not None:
        return short_id
    url_num = self.redis.incr('last-url-id')
    short_id = base36_encode(url_num)
    self.redis.set('url-target:' + short_id, url)
    self.redis.set('reverse-url:' + url, short_id)
    return short_id
```

键 **`reverse-url:` + 实际 url** 会存储 `short_id` 的值。如果 URL 已经提交过了则不会执行操作，仅仅只是返回对应的 short ID 。否则我们将递增 `last-url-id` 这个键的值并将其转换为 base36 。然后我们存储该连接以及其逆解析到 redis 中。这里是转换为 base36 的方法：

``` python
def base36_encode(number):
    assert number >= 0, 'positive integer required'
    if number == 0:
        return '0'
    base36 = []
    while number != 0:
        number, i = divmod(number, 36)
        base36.append('0123456789abcdefghijklmnopqrstuvwxyz'[i])
    return ''.join(reversed(base36))
```

现在我们这个视图欠缺的只是一个 template 了。我们稍后会创建它，不过首先编写其他视图，然后再一次过编写所有的 template 。


## 第六步：重定向视图

重定向视图非常简单，它只需要查找 redis 中的链接并重定向过去就完事了。除此之外，我们还需要累加一个计数器，以便我们知道这个链接被点击了多少次：

``` python
def on_follow_short_link(self, request, short_id):
    link_target = self.redis.get('url-target:' + short_id)
    if link_target is None:
        raise NotFound()
    self.redis.incr('click-count:' + short_id)
    return redirect(link_target)
```

如果找不到 URL 则手动抛出 `NotFound` 异常。这个异常会被冒泡到 `dispatch_request` 函数并转换为一个默认的 404 应答。

## 第七步：详情视图

详情视图与上述重定向视图类似，我们只需要重新渲染一下模板即可。除此之外，我们还要求 redis 查看链接被点击的次数，并且如果这个键不存在则默认将其值设为 0 ：

``` python
def on_short_link_details(self, request, short_id):
    link_target = self.redis.get('url-target:' + short_id)
    if link_target is None:
        raise NotFound()
    click_count = int(self.redis.get('click-count:' + short_id) or 0)
    return self.render_template('short_link_details.html',
                                link_target=link_target,
                                short_id=short_id,
                                click_count=click_count
                                )
```

> 因为 redis 里存放的都是字符串，所以你需要手动将其转换为 `int` 类型。

## 第八步：模板

以下是所有模板。你只要将它们放入 *templates* 目录下即可。Jinja2 支持模板继承，因此我们要做的第一件事就是创建一个使用占位符表示 *block* 的布局模板(layout template)。我们还设置了 Jinja2 来自动使用 HTML 规则的转义字符串，这可以防止 XSS 攻击以及渲染错误。

全部文件详见 *templates* 目录。

## 第九步：样式

白纸黑字始终不太美观，所以我们应用了一个简单的样式表。

详见 *static/style.css* 文件内容。