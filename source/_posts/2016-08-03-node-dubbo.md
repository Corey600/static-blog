---
layout: post
title: 使用node开发dubbo远程调用客户端
category : 备忘
tagline: "备忘"
tags : [前端,node,dubbo]
excerpt_separator: <!--more-->
---

### 简介

>DUBBO是一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，是阿里巴巴SOA服务化治理方案的核心框架，每天为2,000+个服务提供3,000,000,000+次访问量支持，并被广泛应用于阿里巴巴集团的各成员站点。

#### 基本架构如下：

![architecture](/images/20160803/dubbo-architecture.jpg)

其中 Monitor 和 Container 暂时略过不讲：

- Provider: 暴露服务的服务提供方。
- Consumer: 调用远程服务的服务消费方。
- Registry: 服务注册与发现的注册中心。
- * Monitor: 统计服务的调用次调和调用时间的监控中心。
- * Container: 服务运行容器。

<!--more-->

#### 调用关系说明

1. 服务提供者在启动时，向注册中心注册自己提供的服务。
2. 服务消费者在启动时，向注册中心订阅自己所需的服务。
3. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
4. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。

_参考链接_ : [dubbo架构](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-%E6%9E%B6%E6%9E%84)

### 部署dubbo环境

系统环境需要 jdk1.7，如果是 centos，默认安装的是 openJDK，需要卸载后重新安装 jdk1.7。

_下载链接_ : [zookeeper](http://mirrors.hust.edu.cn/apache/zookeeper/zookeeper-3.4.8/zookeeper-3.4.8.tar.gz) / [tomcat](http://apache.fayea.com/tomcat/tomcat-6/v6.0.45/bin/apache-tomcat-6.0.45.tar.gz)

#### zookeeper注册中心

解压安装

```
tar zxvf zookeeper-3.4.8.tar.gz
cd zookeeper-3.4.8
```

创建配置文件

```
vim conf/zoo.cfg
```

修改zoo.cfg的内容如下

```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
```

启动

```
./bin/zkServer.sh start
```

停止

```
./bin/zkServer.sh stop
```

#### 使用tomcat启动管理控制台

解压安装tomcat，并删除默认的 webapps/ROOT 目录

```
tar zxvf apache-tomcat-6.0.45.tar.gz
cd apache-tomcat-6.0.45
rm -rf webapps/ROOT
```

将dubbo管理控制台包解压到 webapps/ROOT 目录

```
unzip dubbo-admin-2.4.1.war -d webapps/ROOT
```

修改配置文件 webapps/ROOT/WEB-INF/dubbo.properties 为如下内容，因为和 zookeeper 注册中心部署在同一台机器，这里配置为本地地址 `zookeeper://127.0.0.1:2181` 就可以了

```
dubbo.registry.address=zookeeper://127.0.0.1:2181
dubbo.admin.root.password=root
dubbo.admin.guest.password=guest
```

启动

```
./bin/startup.sh
```

停止

```
./bin/shutdown.sh
```

访问 `http://127.0.0.1:8080/`: (用户:root,密码:root 或 用户:guest,密码:guest)

### 简单的提供者和消费者

#### 服务提供者

定义服务接口: (该接口需单独打包，在服务提供方和消费方共享)

```java
// DemoService.java
package com.alibaba.dubbo.demo;
public interface DemoService {
    String sayHello(String name);
}
```

在服务提供方实现接口：(对服务消费方隐藏实现)

```java
// DemoServiceImpl.java
package com.alibaba.dubbo.demo.provider;
import com.alibaba.dubbo.demo.DemoService;
public class DemoServiceImpl implements DemoService {
    public String sayHello(String name) {
        return "Hello " + name;
    }
}
```

用Spring配置声明暴露服务：

```xml
<!-- provider.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://code.alibabatech.com/schema/dubbo
    http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
 
    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="hello-world-app"  />
 
    <!-- 使用multicast广播注册中心暴露服务地址 -->
    <dubbo:registry address="multicast://224.5.6.7:1234" />
 
    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />
 
    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService" />
 
    <!-- 和本地bean一样实现服务 -->
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl" />
 
</beans>
```

加载Spring配置：

```java
// Provider.java
import org.springframework.context.support.ClassPathXmlApplicationContext;
public class Provider {
    public static void main(String[] args) throws Exception {
        String configLocation = "spring/dubbo-provider.xml";
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(configLocation);
        context.start();
        System.in.read(); // 按任意键退出
    }
}
```

#### 服务消费者

通过Spring配置引用远程服务：

```xml
<!-- consumer.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://code.alibabatech.com/schema/dubbo
    http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
 
    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
    <dubbo:application name="consumer-of-helloworld-app"  />
 
    <!-- 使用multicast广播注册中心暴露发现服务地址 -->
    <dubbo:registry address="multicast://224.5.6.7:1234" />
 
    <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
    <dubbo:reference id="demoService" interface="com.alibaba.dubbo.demo.DemoService" />
 
</beans>
```

加载Spring配置，并调用远程服务：(也可以使用IoC注入)

```java
// Consumer.java
import org.springframework.context.support.ClassPathXmlApplicationContext;
import com.alibaba.dubbo.demo.DemoService;
public class Consumer {
    public static void main(String[] args) throws Exception {
        String configLocation = "spring/dubbo-consumer.xml";
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(configLocation);
        context.start();
        DemoService demoService = (DemoService)context.getBean("demoService"); // 获取 远程 服务代理
        String hello = demoService.sayHello("world"); // 执行远程方法
        System.out.println( hello ); // 显示调用结果
    }
}
```

_参考链接_ : [Dubbo实例](http://www.tuicool.com/articles/bYfIBfU)

### node客户端设计

#### 整体流程

![dubbo](/images/20160803/dubbo.png)

#### 编解码及序列化

###### 发送的传输协议格式

协议头为定长 16个字节（128位）

 *  0 - 1B dubbo协议魔数(short) 固定为 0xda 0xbb
 *  2 - 2B 消息标志位
 *  3 - 3B 状态位
 *  4 -11B 设置消息的id long类型
 * 12 -15B 设置消息体body长度 int类型

协议体为hessian2序列化后的数据，数据值按顺序依次为

 *  1. dubbo的版本信息
 *  2. 服务接口名
 *  3. 服务的版本号
 *  4. 调服务的方法名
 *  5. 调服务的方法的参数描述符
 *  6. 遍历传输的参数值逐个序列化
 *  7. 将整个附属信息map对象attachments序列化

其中，方法的参数必须符合 java 类型表示的方法，具体可以参考模块说明： [js-to-java](https://github.com/node-modules/js-to-java)


###### 接收的传输协议格式

![parse](/images/20160803/dubbo_protocol_header.jpg)

其中 16-20位为序列化协议的类型ID，hessian2 的协议ID为 2，node客户端目前不支持其他的序列化协议。

其中 24-31未为状态位，具体值表示如下：

| value | msg |
|:-:|:-:|
| 20 | 'ok'  |
| 30 | 'clien side timeout'  |
| 31 | 'server side timeout'  |
| 40 | 'request format error'  |
| 50 | 'response format error'  |
| 60 | 'service not found'  |
| 70 | 'service error' |
| 80 | 'internal server error' |
| 90 | 'internal server error' |

协议体为hessian序列化后的数据，需要反序列化后读取。第一个读取的数据为整数，方法表示返回值类型，具体值表示如下：

```
// 结果标志位为数据段第一个字节，为 0x90 0x91 0x92其中之一
// 实际为hessian协议的简化整数表示，反序列化后为： 0 1 2
RESPONSE_WITH_EXCEPTION = 0; // 返回的是java异常
RESPONSE_VALUE = 1; // 返回方法的返回值
RESPONSE_NULL_VALUE = 2; // 该方法没有返回值
```

如果有，随后读取的就是具体的返回值数据，也符合 java 类型表示的方法。

### 客户端实现源码

[https://github.com/Corey600/zoodubbo](https://github.com/Corey600/zoodubbo)

### Todo List

1. 缓存服务列表，监听变化更新列表，不再每次都重新获取
2. 使用连接池管理连接，添加负载均衡策略
3. 累计调用次数和调用时间，定时发送统计数据到监控中心