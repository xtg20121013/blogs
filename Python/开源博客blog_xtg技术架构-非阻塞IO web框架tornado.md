>[blog_xtg](https://github.com/xtg20121013/blog_xtg)是我个人写的一个开源博客，其web框架使用的就是tornado，以下这篇博客就简单介绍下tornado在blog_xtg中的使用。

######Tornado简介
Tornado 是一个Python web框架和异步网络库，起初由 FriendFeed(后该公司被facebook收购，目前tornado由facebook开发维护)开发。由于非阻塞的特性，他在处理长连接、websocket等保持连接时间较长的请求时，并发能力很强。


Tornado 大体上可以被分为4个主要的部分:

- web框架 (包括创建web应用的 RequestHandler 类，还有很多其他支持的类).
- HTTP的客户端和服务端实现 (HTTPServer and AsyncHTTPClient).
- 异步网络库 (IOLoop and IOStream), 为HTTP组件提供构建模块，也可以用来实现其他协议.
- 协程库 (tornado.gen) 允许异步代码写的像同步代码一样直观，而不用链式回调的方式.

我之所以选用tornado，主要因为目前非阻塞IO的框架在近几年的web技术中特别火，比如node.js，而tornado依靠底层基于epoll(Linux)或者kqueue(BSD和MAC OSX)的IOLoop实现非阻塞IO，而且经过FriendFeed的实践，已经证明他是绝对优秀可靠的非阻塞IO框架，再加上他协程特性，让基于他的异步代码可以像阻塞多线程框架的同步代码一样易读易维护。

######Tornado在blog_xtg中的使用
目前，我在项目的开发环境中使用的是tornado4.4.1版本。

main.py （通过python main.py [--port 8888] 启动tornado server实例监听对应端口的web请求）

```
# coding=utf-8
import os
import log_config
from config import config
from tornado.options import options
import tornado.ioloop
import concurrent.futures
import controller.home
from tornado.web import url
from extends.session_tornadis import SessionManager
from service.init_service import site_init
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# tornado server相关参数
settings = dict(
    template_path=os.path.join(os.path.dirname(__file__), "template"),
    static_path=os.path.join(os.path.dirname(__file__), "static"),
    compress_response=config['compress_response'],
    xsrf_cookies=config['xsrf_cookies'],
    cookie_secret=config['cookie_secret'],
    login_url=config['login_url'],
    debug=config['debug'],
)


# url映射
handlers = [
    url(r"/", controller.home.HomeHandler, name="index"),
    url(r"/auth/login", controller.home.LoginHandler, name="login"),
    url(r"/([0-9]+)", controller.home.HomeHandler, name="articleTypes"),
    url(r"/auth/logout", controller.home.LogoutHandler, name="logout"),
    url(r"/([0-9]+)", controller.home.HomeHandler, name="articleSources"),
    url(r"/", controller.home.HomeHandler, name="admin.submitArticles"),
    url(r"/", controller.home.HomeHandler, name="admin.account"),
]


# sqlalchemy连接池配置以及生成链接池工厂实例
def db_poll_init():
    engine_config = config['database']['engine_url']
    engine = create_engine(engine_config, **config['database']["engine_setting"])
    db_poll = sessionmaker(bind=engine)
    return db_poll;


# 继承tornado.web.Application类，可以在构造函数里做站点初始化（初始数据库连接池，初始站点配置，初始异步线程池，加载站点缓存等）
class Application(tornado.web.Application):
    def __init__(self):
        super(Application, self).__init__(handlers, **settings)
        self.session_manager = SessionManager(config['redis_session'])
        self.thread_executor = concurrent.futures.ThreadPoolExecutor(config['max_threads_num'])
        self.db_pool = db_poll_init()
        site_init(self.db_pool())

if __name__ == '__main__':
    options.define("port", default=config['default_server_port'], help="run server on a specific port", type=int)
    options.define("console_log", default=False, help="print log to console", type=bool)
    options.define("file_log", default=True, help="print log to file", type=bool)
    options.define("file_log_path", default=log_config.FILE['log_path'], help="path of log_file", type=str)
    options.logging = None
    # 读取 项目启动时，命令行上添加的参数项
    options.parse_command_line()
    # 加载日志管理
    log_config.init(options.port, options.console_log, options.file_log, options.file_log_path)
    Application().listen(options.port);
    tornado.ioloop.IOLoop.current().start()
```

controller.base.py 所有controller的基类，可以写一些所有请求通用的方法（读取保存session，获取登录信息等）

```
# coding=utf-8
import tornado.web
from tornado import gen
from extends.session_tornadis import Session
from config import session_keys
from model.logined_user import LoginUser


class BaseHandler(tornado.web.RequestHandler):

    def initialize(self):
        self.session = None
        self.db_session = None
        self.session_save_tag = False
        self.thread_executor = self.application.thread_executor
        self.async_do = self.thread_executor.submit

    @gen.coroutine
    def prepare(self):
        yield self.init_session()
        if session_keys['login_user'] in self.session:
            self.current_user = LoginUser(self.session[session_keys['login_user']])

    @gen.coroutine
    def init_session(self):
        if not self.session:
            self.session = Session(self)
            yield self.session.init_fetch()

    def save_session(self):
        self.session_save_tag = True
        self.session.generate_session_id()

    @property
    def db(self):
        if not self.db_session:
            self.db_session = self.application.db_pool()
        return self.db_session

    def save_login_user(self, user):
        login_user = LoginUser(None)
        login_user['id'] = user.id
        login_user['name'] = user.username
        login_user['avatar'] = user.avatar
        self.session[session_keys['login_user']] = login_user
        self.current_user = login_user
        self.save_session()

    def logout(self):
        if session_keys['login_user'] in self.session:
            del self.session[session_keys['login_user']]
            self.save_session()
        self.current_user = None

    def has_message(self):
        if session_keys['messages'] in self.session:
            return bool(self.session[session_keys['messages']])
        else:
            return False

    # category:['success','info', 'warning', 'danger']
    def add_message(self, category, message):
        item = {'category': category, 'message': message}
        if session_keys['messages'] in self.session and \
                isinstance(self.session[session_keys['messages']], dict):
            self.session[session_keys['messages']].append(item)
        else:
            self.session[session_keys['messages']] = [item]
        self.save_session()

    def read_messages(self):
        if session_keys['messages'] in self.session:
            all_messages = self.session.pop(session_keys['messages'], None)
            self.save_session()
            return all_messages
        return None

    @gen.coroutine
    def on_finish(self):
        if self.db_session:
            self.db_session.close()
            print "db_info:", self.application.db_pool.kw['bind'].pool.status()
        if self.session is not None and self.session_save_tag:
            yield self.session.save()

```
tornado.web.RequestHandler的生命周期是initialize() -> prepare() -> get()/post() -> on_finish()

以下方法可以重载:

1. initialize() 在构造函数后调用，一般用于定义参数，不可异步

2. prepare() 在具体的get()/post()/put()/delete()等执行前调用，一般用于加载登录信息，过滤请求等，可异步

3. get()/post() 处理请求

4. on_finish() 请求后的清理，保存缓存，session等,该方法不能传递任何数据到客户端，所以不能操作cookie，可异步

5. get_current_user() 第一次使用实例中的current_user参数且为None时会调用该方法，因为不能异步，所以需要通过异步来获取的，请写在prepare()中，blog_xtg便是在prepare()中异步从redis读取。


controller.home.py 具体的请求处理

```
# coding=utf-8
from tornado import gen

from base import BaseHandler
from service.user_service import UserService


class HomeHandler(BaseHandler):

    def get(self):
        self.render("index.html")


class LoginHandler(BaseHandler):

    def get(self):
        next_url = self.get_argument('next', '/')
        self.render("auth/login.html", next_url=next_url)

    @gen.coroutine
    def post(self):
        username = self.get_argument('username')
        password = self.get_argument('password')
        next_url = self.get_argument('next', '/')
        user = yield self.async_do(UserService.get_user, self.db, username)
        if user is not None and user.password == password:
            self.save_login_user(user)
            self.add_message('success', u'登陆成功！欢迎回来，{0}!'.format(username))
            self.redirect(next_url)
        else:
            self.add_message('danger', u'登陆失败！用户名或密码错误，请重新登陆。')
            self.get()


class LogoutHandler(BaseHandler):

    def get(self):
        self.logout()
        self.add_message('success', u'您已退出登陆。')
        self.redirect("/")

```


至此，就是整个tornado在启动、处理请求的完整代码。(ps:配置很简单，代码也很少，比起java spring mvc 配置一大堆的xml简便多了，不过这也是python吸引我的一个地方)

######Tornado多实例部署可能会遇到的问题
tornado不仅仅是一个web framework，他还是一个简易的web server，这让他可以直接作为一个server来接收处理http请求，而不需要依靠wsgi容器。但是这个webserver过于简单，只支持单进程，所以官方推荐的方式是多进程的范式，启动多个tornado server实例分别监听不同端口，在上层通过类似nginx的成熟高效的http server来做负载均衡，将请求转发到合适端口的tornado实例中。（[参考tornado官方文档运行部署篇](http://www.tornadoweb.org/en/stable/guide/running.html)）

而blog_xtg设计之初就准备基于该方式部署生产环境。但是多进程多实例的部署方式使得整个应用服务器的架构设计必须满足可分布式部署的要求，所以需要满足以下几点：

-  多实例间的日志不冲突。
-  多实例间的缓存同步。
-  多实例间的session同步。

以上的几点，我都会在以后的博客中给出我在blog_xtg的解决方案。


附:

blog_xtg开源博客的github地址 [https://github.com/xtg20121013/blog_xtg](https://github.com/xtg20121013/blog_xtg)

tonado最新版官方文档 [http://www.tornadoweb.org/en/stable/guide.html](http://www.tornadoweb.org/en/stable/guide.html)

tonado中文文档(版本可能落后) [http://tornado.moelove.info/zh/latest/guide.html](http://tornado.moelove.info/zh/latest/guide.html)

