>上一篇（一）已经构建好了docker环境及持续集成所涉及的工具jenkins、maven，这篇博文会介绍接下来的搭建过程

######一. 构建自己项目的镜像的Dockerfile
我的项目是个java项目(名叫ydb)，Servlet容器是tomcat，外层套个nginx做动静分离，用supervisor做进程管理。

项目镜像是基于Debian8，官方apt源比较慢，所以我换了阿里和网易的源

sources.list

```
deb http://mirrors.aliyun.com/debian/ jessie main non-free contrib
deb http://mirrors.aliyun.com/debian/ jessie-proposed-updates main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ jessie main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ jessie-proposed-updates main non-free contrib
deb http://mirrors.163.com/debian/ jessie main non-free contrib
deb http://mirrors.163.com/debian/ jessie-updates main non-free contrib
deb http://mirrors.163.com/debian/ jessie-backports main non-free contrib
deb-src http://mirrors.163.com/debian/ jessie main non-free contrib
deb-src http://mirrors.163.com/debian/ jessie-updates main non-free contrib
deb-src http://mirrors.163.com/debian/ jessie-backports main non-free contrib
deb http://mirrors.163.com/debian-security/ jessie/updates main non-free contrib
deb-src http://mirrors.163.com/debian-security/ jessie/updates main non-free contrib

```


项目基础镜像Dockerfile如下

```
# MAINTAINER        mhq <maohangqing@c8yun.com>
# DOCKER-VERSION    1.12.3
#
# Dockerizing ydb_app: Dockerfile for building ydb_app images
#
FROM tomcat:7.0.73-jre7
MAINTAINER mhq <maohangqing@c8yun.com>

ENV JAVA_OPTS "$JAVA_OPTS -Djava.security.egd=file:/dev/./urandom"
ENV NGINX_VERSION 1.11.6-1~jessie
ENV PYTHON_VERSION 2.7.9-1
ENV SUPERVISOR_VERSION 3.3.1

ADD sources.list /etc/apt/sources.list 
RUN rm -rf /etc/apt/sources.list.d/* 

# nignx
RUN apt-key adv --keyserver hkp://pgp.mit.edu:80 --recv-keys 573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62 \
  && echo "deb http://nginx.org/packages/mainline/debian/ jessie nginx" >> /etc/apt/sources.list \
  && apt-get -o Acquire::Check-Valid-Until=false update \
	&& apt-get install --no-install-recommends --no-install-suggests -y \
						ca-certificates \
						nginx=${NGINX_VERSION} \
						nginx-module-xslt \
						nginx-module-geoip \
						nginx-module-image-filter \
						nginx-module-perl \
						nginx-module-njs \
						gettext-base \
	&& rm -rf /var/lib/apt/lists/*

# nginx log
# RUN ln -sf /dev/stdout /var/log/nginx/access.log \
# 	&& ln -sf /dev/stderr /var/log/nginx/error.log

# python and pip
RUN apt-get -o Acquire::Check-Valid-Until=false update \
  && apt-get install --force-yes -y python=${PYTHON_VERSION} \
  && wget https://bootstrap.pypa.io/get-pip.py \
  && python get-pip.py \
  && rm get-pip.py

# supervisor
RUN pip install supervisor==${SUPERVISOR_VERSION} \
  && mkdir -p /var/log/supervisor \
  && mkdir -p /home/shunova

#时区问题
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# do something to start the app
ONBUILD RUN rm -rf /usr/local/tomcat/webapps/*
ONBUILD ADD app.war /home/shunova/apps/
ONBUILD COPY tomcat_server.xml /usr/local/tomcat/conf/server.xml
ONBUILD COPY nginx.conf /etc/nginx/nginx.conf
ONBUILD COPY supervisord.conf /etc/supervisord.conf



EXPOSE 80 443

CMD ["supervisord", "-n"]
```

构建该镜像并上传到自己的私有仓库（假如你的镜像仓库地址是registry.xxx.com:5000）

```
docker build -t registry.xxx.com:5000/ydb/ydb_ori:1.0 .
docker push registry.xxx.com:5000/ydb/ydb_ori:1.0
```

用于持续集成自动构建项目镜像的Dockerfile，分成两个镜像是避免一些重复的构建过程，加快构建时间。其实这个Dockerfile就只是FROM刚才创建的基础镜像，不过ONBUILD的语句会在这个镜像构建时执行。


```
#
# MAINTAINER        mhq <maohangqing@c8yun.com>
# DOCKER-VERSION    1.12.3
#
# Dockerizing ydb_app: Dockerfile for building ydb_app images
#
# these ONBUILD cmds in ydb_ori:1.0
# ONBUILD ADD app.war /home/shunova/
# ONBUILD COPY tomcat_server.xml /usr/local/tomcat/conf/server.xml
# ONBUILD COPY nginx.conf /etc/nginx/nginx.conf
# ONBUILD COPY supervisord.conf /etc/supervisord.conf

FROM registry.xxx.com:5000/ydb/ydb_ori:1.0
MAINTAINER mhq <maohangqing@c8yun.com>
```


######二. 在jenkins中创建项目,构建测试站点的自动化部署项目
1. 输入项目名，选择<构建一个自由风格的软件项目>。
2. 源码管理配置
	
	我们的项目是用git管理的，所以选择Git。所以这么填：
	
	![源码管理配置](http://upload-images.jianshu.io/upload_images/3298892-26f106fd580798fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 构建配置
	选择 添加构建步骤-Execute shel
	填入构建脚本 例如：
	![构建](http://upload-images.jianshu.io/upload_images/3298892-e17171710647cc6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
	```
REGISTRY_URL=registry.xxxx.com:5000
#用构建号来生成镜像的版本号
YDB_IMAGE_TAG=${REGISTRY_URL}/ydb/ydb_app:1.0.${BUILD_NUMBER}
#通过maven镜像编译
docker run --rm -v /home/docker/maven/.m2:/root/.m2 -v /home/docker/jenkins_home/workspace/${JOB_NAME}:/usr/src/mymaven -w /usr/src/mymaven maven:3.3.9-jdk-7 mvn clean install
#通过Dockerfile构建新项目镜像
cp -rf ${WORKSPACE}/target/ydb-web.war ${WORKSPACE}/docker_build/app.war
cd ${WORKSPACE}/docker_build
docker build -t ${YDB_IMAGE_TAG} .
#启动新版本ydb_app容器
if docker ps -a | grep -i ydb_app; then
	docker rm -f ydb_app
fi
#这个容器的数据库是连接同个宿主机上另外两个容器（mysql，mongo）,并且项目中可以通过环境变量来获取数据库连接参数，所以这里设置了这些环境变量
docker run -d -p 80:80 -p 443:443 --link mysql:mysql --link mongo:mongo \
            -e YDB_MYSQL_URL=jdbc:mysql://mysql:3306/ydb.new?characterEncoding=utf-8 \
            -e YDB_MYSQL_USER=xxx \
            -e YDB_MYSQL_PASSWORD=xx \
            -e YDB_MONGO_HOST=xx \
            -e YDB_MONGO_USER=xxx \
            -e YDB_MONGO_PASSWORD=xxx \
            -e JAVA_OPTS="$JAVA_OPTS -server -Xms1024m -Xmx3072m -XX:PermSize=256m  -XX:MaxPermSize=512m -XX:+UseParallelOldGC -XX:+ScavengeBeforeFullGC -XX:+TieredCompilation " \
            -v /home/docker/apps/test_ydb_certs:/certs \
            --name ydb_app ${YDB_IMAGE_TAG}
	```

4. 至此，测试环境自动部署已经完成。保存返回jenkins主菜单，点击刚才新建的项目立即构建，看看是否构建成功，是否可以成功访问到自己的项目。
	![立即构建](http://upload-images.jianshu.io/upload_images/3298892-ce9ea8b5328a9ceb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
5. 当然我们的持续集成还不够自动化，我们喜欢提交完代码后立刻就发布到测试站上。所以我们还要配置一下git的webhooks，我们项目仓库用的是gogs，所以很方便，jenkins里装个Gogs plugin插件，然后在gogs中项目设置web钩子（就是webhooks），job=后面填jenkins里创建的项目名。
	![web钩子](http://upload-images.jianshu.io/upload_images/3298892-ef413fcb1b9ab96f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	之后每次提交就会自动触发构建。
	
	如果你们用的不是gogs也可以jenkins搜索你们git仓库插件，或者配合jenkins项目配置里的构建-触发远程构建
	![构建触发器](http://upload-images.jianshu.io/upload_images/3298892-3180a829bfb15208.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
6. OK，现在已经完成了从代码提交-项目编译构建-发布测试镜像的自动化过程，接下来只要测试通过，将通过的镜像push到仓库，然后发布到生产环境即可。当然这一过程需要自己写脚本完成。脚本内容简单点么就用docker-compose -H 到生产环境应用集群中每台服务器启动对应版本的镜像容器。如果生产环境的集群比较大，也可以搭建swarm集群。这里的内容以后有机会再谈吧。