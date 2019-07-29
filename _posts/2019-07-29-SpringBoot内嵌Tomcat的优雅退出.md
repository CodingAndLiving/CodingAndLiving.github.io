---
layout:     post
title:      SpringBoot内嵌Tomcat的优雅退出
subtitle:   SpringBoot内嵌Tomcat的优雅退出
date:       2019-07-29
author:     CodingAndLiving
header-img: img/com01.jpg
catalog: true
tags:
    - Tomcat
    - SpringBoot
    - 优雅退出
---
# 前言

> 在[Linux服务器部署](https://codingandliving.github.io/2019/07/28/%E6%9C%8D%E5%8A%A1%E5%99%A8%E9%83%A8%E7%BD%B2/)一文中讲到了单个Tomcat程序的优化配置，那么针对SpringBoot内嵌的Tomcat，又该如何操作呢？

# 思路

根据*在Linux服务器部署一文中讲到了单个Tomcat程序的优化配置*来看，我们主要实现以下几点：

1. Tomcat的延迟关闭，又名为优雅退出。

2. Tomcat的Connector类型修改为Apr。

# 实现SpringBoot内嵌Tomcat的延迟关闭

#### 思路：

  首先，我们需要在关闭时候，让Tomcat停止接收外部请求，并且处理完已接收的请求，这样子，就可以达到优雅退出的效果。

#### 实现方法：

- 主要关注两个接口 **TomcatConnectorCustomizer** 和 ** ApplicationListener<ContextClosedEvent>**

    1. 前者，是Tomcat的Connector回调接口，至于connector是什么，可以关注我的另外一篇讲解Tomcat的博文

    2. 后者，是监听spring上下文关闭事件的接口

  代码案例，见如下：

    

    @Slf4j
    public class CustomShutdown implements TomcatConnectorCustomizer, ApplicationListener<ContextClosedEvent> {

        private static final int TIMEOUT = 30;

        private volatile Connector connector;

        @Override
        public void onApplicationEvent(ContextClosedEvent event) {
            this.connector.pause();
            Executor executor = this.connector.getProtocolHandler().getExecutor();
            if (executor instanceof ThreadPoolExecutor) {
                try {
                    log.warn("WEB 应用准备关闭");
                    ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;
                    threadPoolExecutor.shutdown(); 
                    if (!threadPoolExecutor.awaitTermination(TIMEOUT, TimeUnit.SECONDS)) {
                        log.warn("WEB 应用等待关闭超过最大时长 " + TIMEOUT + " 秒，将进行强制关闭!");
                        threadPoolExecutor.shutdownNow();
                        if (!threadPoolExecutor.awaitTermination(TIMEOUT, TimeUnit.SECONDS)) {
                            log.error("WEB 应用关闭失败!");
                        }
                    }
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                }
            }
        }

        @Override
        public void customize(Connector connector) {
            this.connector = connector;
        }
    }
      


- 接着呢，我们需要将自定义这类添加到SpringBoot内嵌的Tomcat里面去，参考下面代码

```

    @Bean
    public CustomShutdown customShutdown() {
        return new CustomShutdown();
    }

    @Bean
    public ConfigurableServletWebServerFactory webServerFactory(final CustomShutdown customShutdown) {
        TomcatServletWebServerFactory tomcatServletWebServerFactory = new TomcatServletWebServerFactory();
        tomcatServletWebServerFactory.addConnectorCustomizers(customShutdown);
        return tomcatServletWebServerFactory;
    }
```

- 上述代码完毕后，我们就可以关闭tomcat了，至于关闭的方式可以采取kill命令，或者利用SpringBoot的actuator特性。

    **1. SpringBoot的actuator特性**

        该功能支持通过post请求关闭应用 ，但是本人不是很喜欢，感觉网络不是太安全。

        实现方式：

        1. 添加依赖

        <dependency>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>


        2. 在application文件中，开启支持Post请求关闭应用

        management.endpoint.shutdown.enabled=true
        management.endpoints.web.exposure.include=*

        3. 请求url为 http://localhost:port/actuator/shutdown


        


    **2. kil命令，参考如下**



        kill -15 pid   等同于 kill pid
        这条命令发信号让进程正常退出。所谓的正常退出是指按应用程序自己的退出流程完成退出，这样就可以清理并释放资源。比如 vim 程序，
        如果是正常的退出，就会删除掉临时文件 *.swp。
        既然信号 15 是退出进程的正确方式，那它也应该是最常用的方式，因而我们可以省略参数 -15。

        kill -9 pid  
        相当于强制退出，这样结束掉的进程不会进行资源的清理工作，所以如果你用它来终结掉 vim 的进程，就会发现临时文件 *.swp 没有被删除。



# 实现SpringBoot内嵌Tomcat的connector修改为apr

1. 首先，根据*在Linux服务器部署一文中讲到了单个Tomcat程序的优化配置*，得先安装好apr相关的包,同样的，文章中说到的其他配置，也可以参考实现。

2. 接着，参考下面代码：

```
@Bean
    public ConfigurableServletWebServerFactory webServerFactory(final CustomShutdown customShutdown) {
        TomcatServletWebServerFactory tomcatServletWebServerFactory = new TomcatServletWebServerFactory();
        tomcatServletWebServerFactory.addConnectorCustomizers(customShutdown);

        //添加aprconnector
         tomcatServletWebServerFactory.setProtocol("org.apache.coyote.http11.Http11AprProtocol");

        return tomcatServletWebServerFactory;

    }


如果需要修改connector的属性，参考下面

@Override
    public void customize(Connector connector) {
        this.connector = connector;
        
        Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
        protocol.setConnectionTimeout();
        protocol.setMaxThreads();
        protocol.setMinSpareThreads();
        protocol.setAcceptCount();
        protocol.setMaxHttpHeaderSize();
        protocol.setDisableUploadTimeout();


        Http11AprProtocol protocol1 = (Http11AprProtocol) connector.getProtocolHandler();
        protocol1.setConnectionTimeout();
        protocol1.setMaxThreads();
        protocol1.setMinSpareThreads();
        protocol1.setAcceptCount();
        protocol1.setMaxHttpHeaderSize();
        protocol1.setDisableUploadTimeout();
    }

```