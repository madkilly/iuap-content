# 对象OID组件概述 #

## 组件简介 ##

在数据持久化时，需要生成全局唯一的对象标识ID，传统的UUID能保证唯一性，但是数据量超大之后，查询效率会下降，所以需要一种保证全局唯一的对象标识生成方案。应用在进行扩展时，需要集群部署，多个JVM中生成的唯一标识需要保证不冲突，全局唯一。

## 解决方案 ##

iuap-oid组件提供对实体唯一标识生成的统一封装，包含普通的UUID、基于Redis的自增ID、和snowflake、uapoid算法等。组件提供统一接口对OID进行生成，通过配置的方式实现对几种策略的切换。同时支持对自定义ID生成方案的扩展。

采用Redis自增方式，支持为不同模块设置不同的起始值。

# 整体设计 #

## 依赖环境 ##
iuap-oid组件支持多种ID生成方式，如UUID、Redis自增、snowflake，UAPOID等，部分依赖Jedis的2.6和数据库驱动。组件的依赖配置如下：

	<dependency>
	  <groupId>com.yonyou.iuap</groupId>
	  <artifactId>iuap-oid</artifactId>
	  <version>${iuap.modules.version}</version>
	</dependency>

## 类型说明 ##

- uuid 原生的UUID生成方式
- redis 采用Redis自增方式，可以设置自增的初始值和模块
- snowflake 根据Twitter的算法修改，需要保证workerid唯一
- uapoid 延用原UAP中唯一标识生成算法，依赖数据库，需要初始化表
- custom 客户自定义的ID生成方式

# 使用说明 #

## 配置和使用方式 ##

**1:在工程的classpath中加入对oid组件的依赖**
	<dependency>
		<groupId>com.yonyou.iuap</groupId>
		<artifactId>iuap-oid</artifactId>
		<version>${iuap.modules.version}</version>
	</dependency>

**2:在属性配置文件中，加入oid的使用类型配置**

	#idtype=uuid
	#idtype=redis
	#idtype=snowflake
	idtype=uapoid
	#idproviderclass=com.yonyou.iuap.persistence.oid.CustomIdProvider

	//idtype为需要使用的ID生成类型，目前包括UUID、redis自增、snowflake、uapoid几种类型

**3:使用redis自增主键时候，需要配置redis对应的文件，请参考cache组件**

	<bean id="redisPool" class="com.yonyou.iuap.cache.redis.RedisPoolFactory"
		scope="prototype" factory-method="createJedisPool">
		<constructor-arg value="${redis.url}" />
	</bean>
	
	<bean id="jedisTemplate" class="org.springside.modules.nosql.redis.JedisTemplate">
		<constructor-arg ref="redisPool"></constructor-arg>
	</bean> 

	//属性文件
	redis.url=direct://172.20.6.48:6379?poolSize=50&poolName=mypool

	idtype=redis
	//可以设置特殊模块自增起始值
	IUAP_PRIMARY_MYMODULE_START_VALUE=10000

**4:使用uapoid类型时候，连接数据库，并初始化数据表，stepSize可根据项目指定，为每次取到本地JVM中自增ID的范围**

 	<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
	<property name="dataSource" ref="dataSource"></property>
    </bean>	

    <bean id="uapOidJdbcService" class="com.yonyou.iuap.persistence.oid.UapOidJdbcService">
	    <!--生产态配置项目中使用的jdbcTemplate和Datasource即可-->
	    <property name="jdbcTemplate" ref="jdbcTemplate" />
	    <property name="stepSize" value="5000" />
    </bean>

	注意:建表和初始化sql语句请参考示例工程，最终生成的为20位的字符串，前八位为用户指定的schema名。

	以mysql为例，其sql脚本如下：

    CREATE TABLE pub_oid(schemacode VARCHAR(8) NOT NULL,oidbase VARCHAR(20) NOT NULL,id VARCHAR(36) NOT NULL,ts TIMESTAMP NULL,PRIMARY KEY (id))ENGINE=InnoDB DEFAULT CHARSET=utf8;

	INSERT INTO pub_oid (schemacode, oidbase, id, ts) VALUES (DATABASE(), '100000000001', uuid(), current_timestamp());

如果是基于iuap-jdbc组件使用id生成，可以在实体类上配置注解(iuap-jdbc的注解)的方式，选择类型，示例如下：

	@Entity
	@Table(name="good_demo")
	public class GoodJdbcDemo extends BaseEntity{
	
		private static final long serialVersionUID = -3773469861533388458L;
	
		@Id
	    @Column(name = "productid")
	    @GeneratedValue(strategy=Stragegy.UUID,moudle="example_demo")
	    private String productid;
	
		... ...
	}

其中的strategy为id的生成策略，请参考Stragegy的枚举值，和idtype基本保持一致。

**5:使用snowflake的方式，需要保证workerid一致**

此种方式使用有限制，需要在启动Tomcat时候指定系统属性，ID生成时候会使用到系统属性，如下：

    String workerIdStr = System.getProperty(OID_WORKID);
    if (workerIdStr == null) {
    	workerIdStr = System.getenv(OID_WORKID);
    }
注意，此场景适用在同一台机器或同一个docker容器中启动单个Tomcat或Java应用，否则可能无法保证workerid不唯一。

**6:代码调用ID生成示例**

	//参数根据选择的不同ID生成类型意义不同，请根据项目需求判断，其中redis的为自增模块号，uapoid为对应的数据库的schema名
	IDGenerator.generateObjectID(param);
	
**7:更多API操作和配置方式，请参考对应的示例工程(DevTool/examples/example\_iuap\_oid)**