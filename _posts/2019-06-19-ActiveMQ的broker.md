---
layout:     post
title:      ActiveMQ的broker
subtitle:   ActiveMQ学习系列
date:       2019-06-19
author:     CodingAndLiving
header-img: img/com01.jpg
catalog: true
tags:
    - ActiveMQ
    - JMS
    - 消息中间件
---
# 前言

> **ActiveMQ学习系列**主要分别讲解消息生产者、消费者和broker的特点与性质，这个系列大概会分为若干篇文章。而**本文**主要讲述broker的特点。

# 设置指定的destination（包含queue和topic）流量控制失效
在该系列的ActiveMQ生产者一文中，讲到针对**异步发送非持久化消息**中可以使用流量控制，在此基础上，也可以指定某些destination不使用流量控制，只需要设置producerFlowControl=false.

案例如下：

```

	<destinationPolicy>
		<policyMap>
			<policyEntries>
				<policyEntry topic="主题名称" producerFlowControl="false"/>
			</policyEntries>
		</policyMap>
	</destinationPolicy>
```


# 生效内存限制
在5版本之后，ActiveMQ增加了一个处理，就是会将非持久化消息存储到临时文件之中，这就会导致内存永远用完，当然可以关闭这功能，
设定为不存储临时文件，直接使用内存，并且流量控制机制下，当内存使用完毕后，会导致生产者不发送消息。只需要设置<vmQueueCursor/>。
同样的，这个设置也是针对destination的，（包含queue和topic）

案例如下：

```
	<policyEntry topic="主题名称" producerFlowControl="true" memoryLimit="1mb">
		<pendingQueuePolicy>
			<vmQueueCursor/>
		</pendingQueuePolicy>
	</policyEntry>

```


# 配置生产者客户端异常
我们之前说过，生产者的send方法会由于broker的原因进入堵塞，但是我们更多时候，希望send方法抛出一个异常，而不是堵塞。

方法1： 设置sendFailIfNoSpace为true，那么当broker空间不足时候，生产者调用send方法会马上抛出异常。

案例如下：

```
	<systemUsage sendFailIfNoSpace="true">
		···
		······
		·········
	</systemUsage>
```

方法2： 在5.3.1版本之后，多了配置项sendFailIfNoSpaceAfterTimeout，该配置项当空间不足时候，不会马上抛出异常，
而是等待若干时间之后，如果还是空间不足，才会抛出异常。

案例如下：

```
	<systemUsage sendFailIfNoSpaceAfterTimeout="3000">
		···
		······
		·········
	</systemUsage>
```

# 设置系统占用
可以指定broker使用的内存空间和磁盘空间的总共可用空间大小。

案例如下：

```
	<systemUsage>
		<systemUsage>
			<memoryUsage>
				<memoryUsage percentOfJvmHeap="60"/>
			</memoryUsage>
			<storeUsage>
				<storeUsage limit="100 gb"/>
			</storeUsage>
			<tempUsage>
				<tempUsage ilmit="50 gb"/>
			</tempUsage>
		</systemUsage>
	</systemUsage>
```

# 持久化方案

- kahaDB 

	这个是ActiveMQ5.4版本之后，默认的持久化方案，并且是基于文件存储的，替代了AMQ，原因是没有AMQ建立索引那么耗费时间

	案例如下：

```
	<persistenceAdapter>
		<kahaDB directory="${activemq.data}/kahadb"/>
	</persistenceAdapter>
```
- AMQ

	基于文件存储

- jdbc

	基于数据库存储，数据库有助于其他平台查看数据，但是数据库的效率没有基于上述两个方案性能高。

	修改步骤：

	1. 将mysql的驱动包放于ActiveMQ的lib目录下
	2. 修改配置文件

		

			<persistenceAdapter>
				<jdbcPersistenceAdapter createTablesOnStartup="true" dataSource="#mysql-ds"/>
			</persistenceAdapter>
		

	3. 在配置文件中broker节点外，新增以下内容


			<bean id="mysql-ds" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
				<property name="driverClassName" value="com.mysql.jdbc.Driver"/>
				<property name="url" value="jdbc:mysql://localhost/activemq?relaxAutoCommit=true"/>
				<property name="username" value=""/>
				<property name="password" value=""/>
				<property name="maxActive" value="200"/>
				<property name="poolPreparedStatements" value="true"/>
			</bean>


	4. 手动在mysql中创建名称为activemq的数据库

- Memory基于内存
	
	即将消息存储于内存中，不使用持久化存储，将broker标签设置persistent="false"即可。

	案例如下：

```
	<broker persistent="false">
	········
	</broker>
```

- jdbc message store with activeMq journal

	在jdbc的基础上做了修改，就是先将消息缓存到文件之中，后续再存储数据库里面。



# 消息定时删除
针对destination的消息定时删除需要配置三个属性

- schedulePeriodForDestinationPurge： 执行清理任务的周期，单位毫秒
- gcInactiveDestinations="true" ： 启用清理功能
- InactiveTimeoutBeforeGC="30000" ： destination超时时间，在规定时间内，无有效订阅，没有入栈的消息

案例如下：

```

	<broker schedulePeriodForDestinationPurge="20000">
		<destinationPolicy>
			<PolicyMap>
				<PolicyEntries>
					<PolicyEntry topic=">" gcInactiveDestinations="true" InactiveTimeoutBeforeGC="30000" />
				</PolicyEntries>
			</PolicyMap>
		</destinationPolicy>
	</broker>
```


# 消费者和生产者占用broker空间分离
默认情况下，消费者和生产者使用broker空间是不分离的，我们也可以设置分开，设置项splitSystemUsageForProducersConsumers="true"。
设置之后，生产者空间：消费者空间=6:4,当然，这个也可以设置，producerSystemUsagePortion = 50 ， consumerSystemUsagePortion = 50 

案例如下：

```
	
	<broker splitSystemUsageForProducersConsumers="true" producerSystemUsagePortion="50" 
	consumerSystemUsagePortion="50">
```

# broker支持的传输协议

支持的协议还是挺多的，tcp udp nio， http 等等。

# broker的集群
对于broker集群问题，官网也有说明，[参考文章（里面的broker cluster一栏目中有说明）](http://activemq.apache.org/clustering.html)