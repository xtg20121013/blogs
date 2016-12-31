>docker可以说是应用层的云计算，通过docker可以把应用打包成镜像，像安装包一样可以快速部署在任意装有docker的环境里，而且可以隔离操作系统环境（再也不用担心生产环境和开发测试环境不一样的问题），可以做到项目快速测试上线。所以我觉得无论开发、测试还是运维都有必要了解docker，因为它或多或少都可以减轻你们的工作量。本人前段时间刚用docker搭建了一套持续集成的环境，借此推荐给大家。

#####一、docker入门
docker的基础教程有很多，可以买些书或者视频看看。

个人推荐：

1. [docker官方文档](https://docs.docker.com/): 英语水平好的可以看，官方的毕竟坑少。
2. [docker hub](https://hub.docker.com/): 官方docker镜像仓库，资源丰富。
3. [希云的docker视频教程](http://study.163.com/course/introduction/1273002.htm#/courseDetail): 希云是提供基于docker的企业级容器和Paas的整体解决方案的，简单的说就是靠docker吃饭的，他们家的视频教程还是比较实用易懂的。
4. [DaoCloud](http://get.daocloud.io/): 也是一家类似于希云的公司，不过他们家有个开放的镜像仓库(docker官方常用的下载量比较高的镜像都有)，里面的镜像速度非常快，如果docker官方的资源下载比较慢的话可以尝试这家的镜像。

#####二、搭建持续集成环境的相关镜像及准备过程
######1. 安装docker
我用的服务器是ubuntu14.04的

官方安装方式:

```
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates
sudo apt-key adv \
               --keyserver hkp://ha.pool.sks-keyservers.net:80 \
               --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
echo "deb https://apt.dockerproject.org/repo ubuntu-trusty main" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update
sudo apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual
sudo apt-get install docker-engine
```

也可以用DaoCloud的安装脚本

`curl -sSL https://get.daocloud.io/docker | sh`

######2. 搭建私有的镜像仓库
拉取官方的registry镜像

`docker pull registry:2.5.1`

docker默认的镜像仓库是要求用https方式的，如果要强制使用http的话可以在/etc/default/docker中添加配置项,10.211.55.11:5000替换为要搭建的私有registry的地址及端口

`DOCKER_OPTS="--insecure-registry 10.211.55.11:5000"`

重启docker服务即可

`service docker restart`

启动registry镜像

`docker run -d -p 5000:5000 --restart=always -v /home/docker/registry/data:/var/lib/registry --name registry registry:2.5.1`

访问一下镜像仓库，能访问成功就好了，例如

`http://10.211.55.11:5000/v2/_catalog`

正式环境建议配置ssl证书，并且添加auth认证。

生成对应用户名密码的存储文件，例如：

`
sh -c "docker run --entrypoint htpasswd registry:2.5.1 -Bbn shunova shunova[registry] > /home/docker/registry/auth/htpasswd"
`

将ssl证书放在某目录下，然后启动registry镜像，例如：

`docker run -d -p 5000:5000 --restart=always -v /home/docker/registry/data:/var/lib/registry -v /home/docker/registry/auth:/auth -v /home/docker/registry/certs:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/xxxx.crt -e REGISTRY_HTTP_TLS_KEY=/certs/xxxxx.key -e REGISTRY_AUTH=htpasswd -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd --name registry registry:2.5.1`

即可使用默认的https方式使用私有镜像

######3. 安装Jenkins
拉取Jenkins镜像

`docker pull jenkins:2.19.4`

这个镜像有点特别，默认用户是jenkins，所以在实际使用的时候可能会有一些权限的问题，可以将jenkins加入docker组。

```
root@iZ2893wjzgyZ:~# vim /etc/sudoers

User privilege specification
root    ALL=(ALL:ALL) ALL
jenkins ALL=(ALL:ALL) ALL

jenkins@iZ2893wjzgyZ:~$ usermod -G docker jenkins
```

或者启动镜像时加入`-u root` 。

启动镜像时还要把docker和docker.sock挂载到镜像里，这样可以在jenkins中使用docker，例如：

`docker run -d -p 8080:8080 --name jenkins --restart=always -v /home/docker/jenkins_home:/var/jenkins_home -v /usr/bin/docker:/usr/bin/docker -v /var/run/docker.sock:/var/run/docker.sock -u root jenkins:2.19.4
`

然后浏览器访问[ip]:8080,完成jenkins的初始化即可。初始化可能会验证身份，会让你到服务器目录下找个文件，输入文件的内容。这个时候要到启动的jenkins容器内去找。

`docker exec -it jenkins bash`

然后查看文件`cat xxxx/xxx`

######4. 安装Maven
`docker pull maven:3.3.9-jdk-7`

build maven项目时可以用如下命令：

`docker run -it --rm -v /home/docker/maven/.m2:/root/.m2 -v /home/docker/maven/hello:/usr/src/mymaven -w /usr/src/mymaven maven:3.3.9-jdk-7 mvn clean install`

至此，基础环境搭建完成，具体搭建持续集成的过程，我会在下篇博文中继续介绍。
#####三、一些小坑和题外话
######1. 时区问题
官方的docker镜像默认时区都是UTC，所以可能会出现docker容器中应用时间与实际时间相差8小时。

解决方法1. 启动镜像时添加`-e TZ="Asia/Shanghai" -v /etc/localtime:/etc/localtime:ro`

`docker run -e TZ="Asia/Shanghai" -v /etc/localtime:/etc/localtime:ro --name=tomcat tomcat:8.0.35-jre8`

解决方法2. 用Dockerfile构建镜像时添加

```
#docker 容器的时区问题 
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

```
######2. 远程连接docker
在docker配置文件/etc/default/docker中添加参数

`DOCKER_OPTS="-H 0.0.0.0:7777 -H tcp://127.0.0.1:2375 -H unix:///var/run/docker.sock"`

即可远程使用`docker -H [ip]:7777 ` 调用。

######3. 题外话
公司里，我的项目组以前git仓库用的另外一个项目组提供的仓库，有一天，那个仓库突然无法访问了，不是我们维护的，短期也无法解决。可是项目开发不能停啊，而且我早就有想自己搭建一个想法，那就正好。用的[gogs](https://github.com/gogits/gogs)，在Docker Hub里也有[gogs的docker镜像]()，这就很愉快了，
一键搞定。

`docker run --restart=always --name=gogs -d -p 3000:3000 -p 22:22 -v /home/gogs/data:/data gogs/gogs:0.9.97`

docker真的很方便实用。