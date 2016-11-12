>[blog_xtg](https://github.com/xtg20121013/blog_xtg)是我个人写的一个开源博客，其web框架使用的就是tornado，以下这篇博客就简单介绍下blog_xtg中的日志管理。

######Tornado中日志管理可能存在的问题
前面介绍tornado的博客中已经说过，当生产环境中运行多个实例tornado的时候，因为是一套代码，可能会把日志写到同一个文件中，造成冲突。

######blog_xtg中的解决方案
我的解决思路，因为生产环境中多个实例监听不同的端口，那么在日志命名时使用端口区分即可。顺便将整个日志管理完善，加上控制台输出、文件输出、文件分割清理等。

log_config.py

```
# coding=utf-8
import logging
import logging.handlers
import tornado.log

FILE=dict(
    level="WARNING",
    log_path="logs/log", # 末尾自动添加 @端口号.txt_日期
    when="D", # 以什么单位分割文件
    interval=1, # 以上面的时间单位，隔几个单位分割文件
    backupCount=30, # 保留多少历史记录文件
    fmt="%(asctime)s - %(name)s - %(filename)s[line:%(lineno)d] - %(levelname)s - %(message)s",
)


def init(port, console_handler=False, file_handler=True, log_path=FILE['log_path']):
    logger = logging.getLogger()
    # 配置控制台输出
    if console_handler:
        channel_console = logging.StreamHandler()
        channel_console.setFormatter(tornado.log.LogFormatter())
        logger.addHandler(channel_console)
    # 配置文件输出
    if file_handler:
        if not log_path:
            log_path = FILE['log_path']
        log_path = log_path+"@"+str(port)+".txt"
        formatter = logging.Formatter(FILE['fmt']);
        channel_file = logging.handlers.TimedRotatingFileHandler(
            filename=log_path,
            when=FILE['when'],
            interval=FILE['interval'],
            backupCount=FILE['backupCount'])
        channel_file.setFormatter(formatter)
        channel_file.setLevel(FILE['level'])
        logger.addHandler(channel_file)
```

main.py

```
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

这样在生产环境可以 

`python main.py [--port=8888|console_log=false|file_log=true|file_log_path=logs/log]`

启动进程监听对应端口，生成的日志就不会冲突。例如:

`python main.py --port=8888`

表示启动tornado server并监听8888端口，同时关闭控制台输出，但保留日志文件输出，且日志文件保存路径在项目目录下的logs/log

附:

blog_xtg开源博客的github地址 [https://github.com/xtg20121013/blog_xtg](https://github.com/xtg20121013/blog_xtg)

tonado最新版官方文档 [http://www.tornadoweb.org/en/stable/guide.html](http://www.tornadoweb.org/en/stable/guide.html)

tonado中文文档(版本可能落后) [http://tornado.moelove.info/zh/latest/guide.html](http://tornado.moelove.info/zh/latest/guide.html)

