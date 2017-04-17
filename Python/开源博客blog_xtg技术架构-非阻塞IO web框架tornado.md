>[blog_xtg](https://github.com/xtg20121013/blog_xtg)是我个人写的一个开源分布式博客，其web框架使用的就是tornado，以下这篇博文就简单介绍下tornado在blog_xtg中的使用。

####Tornado简介
Tornado 是一个Python web框架和异步网络库，起初由 FriendFeed(后该公司被facebook收购，目前tornado由facebook开发维护)开发。由于非阻塞的特性，他在处理Http长连接、websocket等保持连接时间较长的请求时，并发能力很强。


Tornado 大体上可以被分为4个主要的部分:

- web框架 (包括创建web应用的 RequestHandler 类，还有很多其他支持的类).
- HTTP的客户端和服务端实现 (HTTPServer and AsyncHTTPClient).
- 异步网络库 (IOLoop and IOStream), 为HTTP组件提供构建模块，也可以用来实现其他协议.
- 协程库 (tornado.gen) 允许异步代码写的像同步代码一样直观，而不用链式回调的方式.

我之所以选用tornado，主要因为目前非阻塞IO的框架在近几年的web技术中特别火，比如node.js，而tornado依靠底层基于epoll(Linux)或者kqueue(BSD和MAC OSX)的IOLoop实现非阻塞IO，而且经过FriendFeed的实践，已经证明他是绝对优秀可靠的非阻塞IO框架，再加上他协程特性，让基于他的异步代码可以像阻塞多线程框架的同步代码一样易读易维护。

####Tornado在blog_xtg中的使用
目前，我在项目的开发环境中使用的是tornado4.4.1版本。

项目主要的配置已经集中到[config.py](https://github.com/xtg20121013/blog_xtg/blob/master/config.py)中,部分参数可以通过命令行参数修改。

所有的url映射都集中到[url_mapping.py](https://github.com/xtg20121013/blog_xtg/blob/master/url_mapping.py)中。

整个项目的入口的在[main.py](https://github.com/xtg20121013/blog_xtg/blob/master/main.py)。

在blog_xtg项目根目录通过python main.py [--port 8888] 启动tornado server实例监听对应端口的web请求

[controller.base.py](https://github.com/xtg20121013/blog_xtg/blob/master/controller/base.py)中的BaseHandler继承于tornado.web.RequestHandler，是blog_xtg中其他controller的基类，可以写一些所有请求通用的方法（读取保存session，获取登录信息等）

tornado.web.RequestHandler的生命周期是initialize() -> prepare() -> get()/post() -> on_finish()

以下方法可以重载:

1. initialize() 在构造函数后调用，一般用于定义参数，不可异步

2. prepare() 在具体的get()/post()/put()/delete()等执行前调用，一般用于加载登录信息，过滤请求等，可异步

3. get()/post() 处理请求

4. on_finish() 请求后的清理，保存缓存，session等,该方法不能传递任何数据到客户端，所以不能操作cookie，可异步

5. get_current_user() 第一次使用实例中的current_user参数且为None时会调用该方法，因为不能异步，所以需要通过异步来获取的，请写在prepare()中，blog_xtg便是在prepare()中异步从redis读取。

总的来说，tornado的配置很简单，代码也很少，比起java spring mvc 配置一大堆的xml简便多了，不过这也是python的主要优势。

####Tornado多实例部署可能会遇到的问题
#####1. 单线程server
tornado不仅仅是一个web framework，他还是一个简易的web server，这让他可以直接作为一个server来接收处理http请求，而不需要依靠wsgi容器。但是这个webserver过于简单，只支持单进程，所以在生产环境中，官方推荐的多进程多主机部署，启动多个tornado server实例分别监听不同端口，在上层通过类似nginx的成熟高效的http server来做负载均衡，将请求转发到合适端口的tornado实例中。（[参考tornado官方文档运行部署篇](http://www.tornadoweb.org/en/stable/guide/running.html)）

#####2. session
tornado本没有实现session，因为他是解决C10K这类高并发问题，由于cpython，作为一个单线程的server无法利用多核的特性，所以官方推荐多进程多实例甚至多主机部署以此来充分利用多核心来处理高并发，一般跨进程的session同步势必用到第三方工具，所以tornado实现单实例的session没有太大的意义。

blog_xtg中的session是通过cookie+redis来实现的，可以跨进程同步session。


附:

blog_xtg开源博客的github地址 [https://github.com/xtg20121013/blog_xtg](https://github.com/xtg20121013/blog_xtg)

tonado最新版官方文档 [http://www.tornadoweb.org/en/stable/guide.html](http://www.tornadoweb.org/en/stable/guide.html)

tonado中文文档(版本可能落后) [http://tornado.moelove.info/zh/latest/guide.html](http://tornado.moelove.info/zh/latest/guide.html)

