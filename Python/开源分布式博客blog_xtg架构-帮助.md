###一、blog_xtg简介
[blog_xtg](https://github.com/xtg20121013/blog_xtg) 是作者`xiaotaogou`基于[Blog_mini](https://github.com/xpleaf/Blog_mini)重构的个人分布式博客系统。

由于不太擅长前端，所以基本照搬[Blog_mini](https://github.com/xpleaf/Blog_mini)的页面，但是整个后端逻辑都是重写的，以下是与[Blog_mini](https://github.com/xpleaf/Blog_mini)的主要区别：

1. 改用tornado框架，是个基于异步IO的web server。
2. 分布式架构，可以多进程多主机启动server实例，再通过nginx等代理服务器做负载均衡，实现横向扩展提高并发性能。
3. 提高多数主要页面访问性能。对频繁查询的组件（例如博客标题、菜单、公告、访问统计）进行缓存，优化sql查询（多条sql语句合并一次执行、仅查需要的字段，例如搜索博文列表不查博文的具体内容）以提高首页博文等主要页面访问性能。
4. 访问统计改为日pv和日uv。
5. 博文编辑器改为markdown编辑器。
6. 引入alembic管理数据库版本。
7. 可使用docker快速部署。

###二、开源声明

1. [blog_xtg](https://github.com/xtg20121013/blog_xtg)是完全开源的，你可以在GitHub获取他的全部代码，当然也可以对他进行二次开发。
2. 作者不对任何基于[blog_xtg](https://github.com/xtg20121013/blog_xtg)开发的或搭建的项目、站点负责。
3. 如果没有获得作者授权，请勿将其商用。


###三、技术支持
如果你有任何疑问，可以给作者留言:

附：	

- 作者博客：[http://blog.xiaotaogou.site](http://blog.xiaotaogou.site)

- 作者简书：[http://www.jianshu.com/u/dfb6bf87c35e](http://www.jianshu.com/u/dfb6bf87c35e)

- blog_xtg的github地址：[https://github.com/xtg20121013/blog_xtg](https://github.com/xtg20121013/blog_xtg)
