---
layout:     post
title:      SpringBoot自定义starter
subtitle:   SpringBoot自定义starter
date:       2019-08-24
author:     CodingAndLiving
header-img: img/com01.jpg
catalog: true
tags:
    - SpringBoot
    - 组件
    - 自定义starter
---
# 前言

> **本文**主要介绍了Spring Boot自定义starter的基本思路。

# 前提介绍

spring Boot出现的其中一个目的，就是为了简化spring日益庞大繁琐的配置。其中Spring Boot Starter就
体现除了组件化的效果。
例如，我们可以在引入spring-boot-starter-web的时候，去除tomcat，再引入其他的依赖，案例如下

1. 利用组件化的思想，去除tomcat-starter。

		<dependency>
		    <groupId>org.springframework.boot</groupId>
		    <artifactId>spring-boot-starter-web</artifactId>
		    <exclusions>
		        <exclusion>
		            <groupId>org.springframework.boot</groupId>
		            <artifactId>spring-boot-starter-tomcat</artifactId>
		        </exclusion>
		    </exclusions>
		</dependency>

2. 然后，根据需要，单独引入其他依赖包即可。


# Spring boot starter的命名规范

1. 官方的groupId为org.springframework.boot,那么我们自定义的，不要和官方一致就好。

2. 官方的artifactId一般是spring-boot-starter-XXX，我们自定义的，一般按照xxx-spring-boot-starter。

# 自定义Spring Boot Starter的方法

1. 首先，我们需要定义一个配置类，用于获取配置。例如下面这个类，就可以获取到配置了

		@ConfigurationProperties(prefix = HelloProperties.HELLO_PREFIX)
		@Data
		public class HelloProperties {
		    static final String HELLO_PREFIX = "hello";

		    private Person person;
		    private boolean enabled;

		    @Data
		    public static class Person {
		        private String personName;
		        private String personAge;
		    }
		}

		配置文件的参考配置应该如下：
		hello: 
		    enabled: true
		    person: 
		    	person-name: xxx
		    	person-age: 18

2. 接着，我们需要一个类，来根据配置文件的内容，动态的生效并且注册bean对象


		@Configuration
		@EnableConfigurationProperties(HelloProperties.class)
		public class HelloAutoConfiguration {

		    @Bean
		    @ConditionalOnProperty(prefix = "hello.person",name = "person-name",matchIfMissing = false)
		    public Person boss(HelloProperties helloProperties) {
		        return new Boss(helloProperties.getPerson().getPersonName(),helloProperties.getPerson().getPersonAge(),
		                helloProperties.isEnabled());
		    }
		}

		EnableConfigurationProperties该注解的效果是使得HelloProperties类，在没有使用注解Component时候，得以往spring注入对象。
		ConditionalOnProperty该注解的效果是根据条件是否成立，来决定是否注册bean。

3. 完整的代码可以查看我上传到[github的仓库](https://github.com/CodingAndLiving/my-spring-boot-starter)