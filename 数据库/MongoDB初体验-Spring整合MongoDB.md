>上一篇分享了MongoDB的配置文件，现在准备正式在已有项目中使用，完成那个公告配置的需求。

整合Spring和MongoDB需要两个jar包：

1.mongodb官方jdbc驱动 mongo-java-driver

2.spring基于mongo-java-driver的连接池管理和ORM的中间件 spring-data-mongodb

因为公司项目的架构所用的技术比较旧，spring还是3.1.2版本的。看了下截止目前能够支持的最高的spring-data-mongodb版本只有1.3.5了，而我的mongodb是3.2版本的，所以能够兼容的最高的mongo-java-driver只有2.14.2。

#####下面是maven的pom.xml引入的这两个包的依赖：

######spring-data-mongodb
```
<dependency>
	<groupId>org.springframework.data</groupId>
	<artifactId>spring-data-mongodb</artifactId>
	<version>1.3.5.RELEASE</version>
</dependency>
```

######mongo-java-driver
```
<dependency>
	<groupId>org.mongodb</groupId>
	<artifactId>mongo-java-driver</artifactId>
	<version>2.14.2</version>
</dependency>
```

#####Spring配置中添加对应的依赖注入：
######applicationContext.xml
xml标签引用

```
<beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:context="http://www.springframework.org/schema/context"
          xmlns:mongo="http://www.springframework.org/schema/data/mongo"
          xsi:schemaLocation=
          "http://www.springframework.org/schema/context
          http://www.springframework.org/schema/context/spring-context-3.0.xsd
          http://www.springframework.org/schema/data/mongo
          http://www.springframework.org/schema/data/mongo/spring-mongo-1.0.xsd
          http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
```

MonggoDb连接池依赖注入配置(相关配置具体含义请参考底部引用的官方文档)

```
	<bean id="propertyConfigurer"
		class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
		<property name="locations">
			<list>
				<value>classpath:mongo.properties</value>
			</list>
		</property>
	</bean>
	<!--连接池配置-->
	<mongo:mongo host="${mongo.host}" port="${mongo.port}">
		<mongo:options connections-per-host="${mongo.options.connections-per-host}"
					   threads-allowed-to-block-for-connection-multiplier="${mongo.options.threads-allowed-to-block-for-connection-multiplier}"
					   connect-timeout="${mongo.options.connect-timeout}"
					   max-wait-time="${mongo.options.max-wait-time}"
					   auto-connect-retry="${mongo.options.auto-connect-retry}"
					   socket-keep-alive="${mongo.options.socket-keep-alive}"
					   socket-timeout="${mongo.options.socket-timeout}"
					   slave-ok="${mongo.options.slave-ok}"
					   write-number="${mongo.options.write-number}"
					   write-timeout="${mongo.options.write-timeout}"
					   write-fsync="${mongo.options.write-fsync}"/>
	</mongo:mongo>
	<!--连接池工厂配置-->
	<mongo:db-factory dbname="${mongo.dbname}" username="${mongo.username}" password="${mongo.password}" mongo-ref="mongo"/>
	<bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate">
		<constructor-arg name="mongoDbFactory" ref="mongoDbFactory"/>
	</bean>
	<!--实体映射自动扫描注入的包-->
	<mongo:mapping-converter>
		<mongo:custom-converters base-package="com.shunova.core.entity.mongo" />
	</mongo:mapping-converter>
```

######mongo.properties

```
mongo.host=localhost
mongo.port=27017
mongo.dbname=dbname
mongo.username=username
mongo.password=password
mongo.options.connections-per-host=20
mongo.options.threads-allowed-to-block-for-connection-multiplier=4
mongo.options.connect-timeout=1000
mongo.options.max-wait-time=1500
mongo.options.auto-connect-retry=false
mongo.options.socket-keep-alive=true
mongo.options.socket-timeout=1500
mongo.options.slave-ok=false
mongo.options.write-number=1
mongo.options.write-timeout=0
mongo.options.write-fsync=true
```

#####实体类：
######项目需求中的公告实体类，Notice
```
package com.shunova.core.entity.mongo;

import org.springframework.data.annotation.Id;
import org.springframework.data.annotation.PersistenceConstructor;
import org.springframework.data.mongodb.core.index.IndexDirection;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

import java.util.Date;

/**
 * Created by mhq on 16/10/28.
 */
@Document
public class Notice {

    public Notice(){}

    @PersistenceConstructor
    public Notice(String id, int siteId, String creator, String title, String content,
                  Date createTime, Date updateTime){
        this.id = id;
        this.siteId = siteId;
        this.creator = creator;
        this.title = title;
        this.content = content;
        this.createTime = createTime;
        this.updateTime = updateTime;
    }

    @Id
    private String id;

    @Indexed
    private int siteId;

    private String creator;

    private String title;

    private String content;

    @Indexed(direction = IndexDirection.DESCENDING)
    private Date createTime;

    private Date updateTime;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public int getSiteId() {
        return siteId;
    }

    public void setSiteId(int siteId) {
        this.siteId = siteId;
    }

    public String getCreator() {
        return creator;
    }

    public void setCreator(String creator) {
        this.creator = creator;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public Date getCreateTime() {
        return createTime;
    }

    public void setCreateTime(Date createTime) {
        this.createTime = createTime;
    }

    public Date getUpdateTime() {
        return updateTime;
    }

    public void setUpdateTime(Date updateTime) {
        this.updateTime = updateTime;
    }
}

```

######相关实体类注解的解释
```
@Id - 文档的唯一标识，在mongodb中为ObjectId，它是唯一的，通过时间戳+机器标识+进程ID+自增计数器（确保同一秒内产生的Id不会冲突）构成。

@Document - 把一个java类声明为mongodb的文档，可以通过collection参数指定这个类对应的文档。@Document(collection="mongodb") mongodb对应表

@DBRef - 声明类似于关系数据库的关联关系。ps：暂不支持级联的保存功能，当你在本实例中修改了DERef对象里面的值时，单独保存本实例并不能保存DERef引用的对象，它要另外保存，如下面例子的Person和Account。

@Indexed - 声明该字段需要索引，建索引可以大大的提高查询效率。

@CompoundIndex - 复合索引的声明，建复合索引可以有效地提高多字段的查询效率。

@GeoSpatialIndexed - 声明该字段为地理信息的索引。

@Transient - 映射忽略的字段，该字段不会保存到mongodb。

@PersistenceConstructor - 声明构造函数，作用是把从数据库取出的数据实例化为对象。该构造函数传入的值为从DBObject中取出的数据
```

#####基本的增删改查

######在dao中注入xml中配置的mongoTemplate

MongoOperations是MongoTemplate的父类

```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.mongodb.core.MongoOperations;
```
```
	@Autowired
	protected MongoOperations mongoOperations;
	
```

######增删改查
```
	public String save(int siteId, String creator, Notice notice){
		Date now = new Date();
		notice.setSiteId(siteId);
		notice.setCreator(creator);
		notice.setCreateTime(now);
		notice.setUpdateTime(now);
		mongoOperations.insert(notice);
		return notice.getId();
	}

	public int update(int siteId, String id, Notice notice){
		Query query = new Query();
		query.addCriteria(Criteria.where("id").is(id).and("siteId").is(siteId));
		Update update = new Update();
		update.set("title", notice.getTitle());
		update.set("content", notice.getContent());
		update.set("updateTime", new Date());
		WriteResult result = mongoOperations.updateFirst(query, update, Notice.class);
		return result.getN();
	}

	public Pager page(Pager pager, int siteId, NoticeSearchParams searchParams){
		Query query = new Query();
		query.addCriteria(Criteria.where("siteId").is(siteId));
		if(!StringTool.isNullOrEmpty(searchParams.getTitleKey())){
			query.addCriteria(Criteria.where("title").regex(searchParams.getTitleKey(), "i"));
		}
		if(searchParams.getSortMode() == NoticeSearchParams.SORT_MODE_CREATE_TIME_DESC){
			query.with(new Sort(new Sort.Order(Sort.Direction.DESC, "createTime")));
		}
		int skipCount = (pager.getPageNo() > 0 ? pager.getPageNo() - 1 : 0) * pager.getPageSize();
		query.skip(skipCount);
		query.limit(pager.getPageSize());
		List<Notice> notices = mongoOperations.find(query, Notice.class);
		pager.setResult(notices);
		return pager;
	}

	public Notice get(int siteId, String id){
		Notice notice = mongoOperations.findOne(
				Query.query(Criteria.where("id").is(id).and("siteId").is(siteId)), Notice.class);
		return notice;
	}

	public void remove(int siteId, String id){
		mongoOperations.remove(
				Query.query(Criteria.where("id").is(id).and("siteId").is(siteId)), Notice.class);
	}
```

至此就完成了基本Spring和MongoDB的整合，以上代码都经过实际测试通过。

[本篇内容参考自Spring官方文档Spring Data MongoDB - Reference Documentation](http://docs.spring.io/spring-data/data-mongo/docs/1.3.5.RELEASE/reference/html/)
