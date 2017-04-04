>我们公司涉足电商多年，以前根据业务转型或拓展单独开发过多个类型的电商平台，目前面临旧平台资源无法整合，拓展新业务发开新平台又要重新再写一些逻辑相似的业务代码。所以我们内部急需这样一个整合所有基础服务的内部SOA平台。本博文就是我在架构这个平台的经验分享。

#####一、电商类的内部SOA平台需要做什么
首先，SOA即面向服务的架构，绝大多数电商平台不外乎用户、商品、订单等基础服务，以及一些发送短信、验证码等通用工具类服务。那么这个内部SOA平台的作用就是整合这些资源，由他向上层平台提供服务，同时可以让用户、商品、订单等数据在所有平台内共享。
![基于内部SOA平台的电商平台结构](http://upload-images.jianshu.io/upload_images/3298892-7c046a9df4c8fda1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过这个SOA平台可以将业务分割，交由不同的部门管理维护，其次拓展的新业务可以做到快速开发上线。例如A平台仅需实现自己业务A独特的需求即可，用户、商品、订单等服务可以调用SOA平台的。
#####二、基于RPC的电商平台内部SOA架构及组件选型

下图是我初步设计的SOA平台架构图以及一些组件的选择
![SOA平台的架构图](http://upload-images.jianshu.io/upload_images/3298892-91f8c59215dd5119.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

######1. RPC框架的选择
RPC即远程过程调用，通俗一点就是跨进程跨主机的通信协议，用在SOA平台与上层业务平台之间的通信。
其实传统的webservice、rest服务都是RPC的一种。但是这些基于HTTP+JSON/XML的通信方式效率不高。但是由于他们的通用性以及易开发的特性使得他们更适合对外第三方接口的开发。
作为对内的SOA平台，肯定追求更加高效的通信方式。这类的RPC框架很多，比如阿里的Dubbo、低调庞大的Zeroc ICE、目前很火的Facebook的thrift等。
- **Dubbo**：阿里出品，不过他们家开源的大多是自家弃用的，所以版本迭代几乎没用。有个当当维护的fork版本[Dubbox](https://github.com/dangdangdotcom/dubbox)。我只简单的了解过Dubbo，Dubbo在RPC通信方面主要基于第三方组件的，其实他更是个服务治理框架，类似于SpringCloud，整体更侧重分布式集群节点间的服务发现、注册、负载均衡等服务协调和提高可用性方面，而且他的RPC仅支持Java间的通信，所以我并没有选用。
- **Zeroc ICE**：其实我一开始是想用这个的，因为这篇文章[《不得不承认Zeroc Ice是RPC王者完爆Dubbo,Thrift》](https://wenku.baidu.com/view/c6bcd65ea1c7aa00b42acb50.html)，还买了一本书[《ZeroC Ice权威指南》](http://baike.baidu.com/link?url=BjgFHANOhBbFhLmnNjaQGVdHyztpl5jEz5PUGOwTUAUOKB3Pd15OILASJt871VFoBZ4NIkFtG63_qNks9wqVhr82z-DOEP10Fk7kLNQjOma34ywsPcIZb-dtSo1kK_ixlot5lYXt8oy2FdXlBFpHzq)学习，这本书作者好像就是那篇文章的作者，不晓得这里有没有猫腻，反正测试结果可以参考但不能全信。看完这本书，感觉ICE太庞大太重，ICE可以算RPC+zookeeper+docker，是个RPC到分布式服务注册发现协调到项目发布部署的整体解决方案，这本书200页左右基本只能做个初级入门，在我实际使用的过程中还是有很多问题要去翻官方文档才能解决，虽然文档还是比较全的，但是毕竟Zeroc公司靠基于ICE的咨询培训赚钱，这里的坑很多，什么默认线程数只有1啦等等，这么庞大的项目，想想以后如果再遇到坑，可能找相关代码都要找半天，学习成本太高，一旦入坑后遇到致命问题，如果不能读懂源码就只能放弃整个项目了，不利于以后的项目维护，风险太大，所以我还是及早脱坑了。
- **thrift**：出自Facebook，从网上各种测试也都证明他的速度很快，支持非常多种语言。而且很轻量，很容易集成，代码也相对ICE好读很多，完全满足我对一个RPC框架的所有需求。

######2. 分布式服务发现注册协调
- **zookeeper**：目前最流行、最火、最成熟的服务发现注册的组件了，也是我最熟悉的就选用他了，安利一个java的client([curator](https://github.com/apache/curator))。

类似zookeeper这样的组件还有etcd、Consul，以后有机会在尝试了。

######3. 数据持久化
- **mysql**：传统关系型数据库，主要存储核心的关系行比较强数据，经常要计算的数据，例如用户名密码、商品报价、订单等数据。
- **mongodb**：经典的Nosql代表之一，主要存储一些扩展和字段不定，关联性较弱的数据，例如商品的详细规格各种参数、用户个性爱好等。
- **Fastdfs**：海量小文件存储，[fastdfs](https://github.com/happyfish100/fastdfs)是个国人写的小文件存储文件系统，解决了海量小文件存储查询的性能问题。主要用来存储图片、html等静态小文件。
- **redis**：内存数据库，比memcached更强大，非常适合做session和缓存。而且单个操作是原子性的，所以我还用INCR命令来生成自增ID，比如生成订单号的场景。
- **Elasticsearch**：相较于lucene、sphinx，es可以像数据库一样去使用，非常方便易用，相较于Solr有更好的性能。所以可以把他用在商品搜索这样的场景。

######4. 消息队列
消息队列可以做异步处理、解耦、流量削锋、日志转发。
- **RocketMQ**： 市场上的消息队列非常多，各种测评也非常多，我们的消息队列应用场景主要是基于发布订阅和其他系统解耦、高峰期堆积消息来做到流量削锋以及日志转发到storm做日志分析。因此我们用消息队列全是实时性要求较低的场景，所以并不追求实时性，但我们更在乎消息的顺序、消息不丢失、消息的堆积能力，综上我们选择了阿里刚捐给Apache的[rocketmq](https://github.com/apache/incubator-rocketmq)。

######5. 日志分析监控

- **Storm**：较为实时的流式日志分析系统。整个过程是集群将日志和性能数据发送到MQ，由Storm去MQ pull下来做实时分析再将分析结果持久化。

######6. 持续集成
- **docker**：docker非常适合做分布式集群的发布部署，基于他做持续集成也是大势所趋，有兴趣的可以看下我的另两篇博文[《基于docker的持续集成（一）》](http://www.jianshu.com/p/e5cb6393a904)、[《基于docker的持续集成（二）》](http://www.jianshu.com/p/219d7477be6b)

因此整体架构如前面架构图所示，各个组件都相对独立而且都是比较成熟的组件，不至于对某个组件依赖太重而被牵制，以后有更好的组件也方便替换。
######7. 开发语言
其实开发语言很随意的，thrift支持多种语言开发server端。
- **Java**：相较于python、php、node等动态脚本语言，Java的开发效率并不高，但正因为他不是动态脚本语言，而是一个纯面向对象的语言，还有一个强大的spring框架，加之Java人才遍地都是，所以Java绝对是最适合多人合作大型应用开发持久维护的语言。因此我们暂定以Java为主。其实对于用Spring管理的项目，applicationContext.xml就是项目最清晰的内部组件目录和项目入口。篇幅有限，具体代码就不贴了，就贴一下我们目前主项目的applicationContext.xml。
 ![applicationContext.xml](http://upload-images.jianshu.io/upload_images/3298892-ec1cdea27da59f73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**bibi一下（仅个人观点）**：其实现在Java界新出了一个很火的框架spring boot以及基于他的服务治理框架spring cloud（类似dubbo）。个人并不是很喜欢spring boot，虽然spring boot简化了xml的配置，减少了开发代码，所有的bean注册都变成了注解或者是默认的约定，但实际上由于配置分散，默认约定过反而也导致了代码可读性降低，定位BUG困难，不利于项目多人参与维护迭代。可以说是在提高开发效率的同时，降低了维护效率，而单纯只为了提高开发效率上，何不选用python、php、node等脚本语言呢。