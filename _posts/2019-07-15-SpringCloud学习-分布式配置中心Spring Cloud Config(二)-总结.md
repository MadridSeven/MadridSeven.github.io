---
layout: post
title: "SpringCloud学习-分布式配置中心Spring Cloud Config(二)-总结"
date: 2019-07-15
description: "Spring，SpringBoot，SpringCloud"
tag: JAVA进阶学习

---

&emsp;&emsp;在上一篇博文中，实现了一个简单的 Spring-Cloud-Config 应用，其中包含了
服务端和客户端两部分，实现了客户端从分布式配置中心获取配置的基本功能，这篇博文，将对
Spring-Cloud-Config 做一个总结，并简单介绍其个性化配置及特殊功能。<font color="red">阅读本文之前，务必阅读上一篇博文！</font>链接：[SpringCloud学习-分布式配置中心Spring Cloud Config(一)-基础](https://madridseven.github.io/2018/01/SpringCloud%E5%AD%A6%E4%B9%A0-%E5%88%86%E5%B8%83%E5%BC%8F%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83Spring-Cloud-Config/)

### 用途

&emsp;&emsp;用来为分布式系统中的基础设施和微服务应用提供集中化的外部配置支持。

### 解决痛点

&emsp;&emsp;在分布式系统或微服务应用中，各个模块或服务之间的配置文件都是独立的，但是大多数时候很多配置都是重复的，这样一来造成了很大的维护成本，而Spring Cloud Config 解决了这个问题。

### 特点

- 分为客户端和服务端两部分
- 服务端也称为分布式配置中心，用来连接配置仓库并为客户端提供获取配置信息、加密/解密信息等访问接口。
- 客户端则是微服务架构中的各个微服务应用，它们通过指定的配置中心来管理应用的配置内容。
- 默认采用 Git 来存储配置信息。

### 依赖

&emsp;&emsp;服务端：
```xml
<dependency>
 <groupId>org.springframework.cloud</groupId>
 <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```
&emsp;&emsp;客户端：
```xml
<dependency>
 <groupId>org.springframework.cloud</groupId>
 <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

### 配置信息的url与配置文件的映射关系

-	/{application}/{profile}[/{label}]
-	/{application}-{profile}.yml
-	/{label}/{application}-{profile}.yml
-	/{application}-{profile}.properties
-	/{label}/{application}-{profile}.properties

### 注解

&emsp;&emsp;开启Spring Cloud Config的服务端功能 @EnableConfigServer

### 基础架构

&emsp;&emsp;下图是一个简单的分布式配置中心和客户端结合的基础架构样例，其中主要包含以下几个要素。

-	远程git仓库：用来存储配置文件
-	Config Server：服务端，也就是分布式配置中心，在该工程中指定所要连接的Git 仓库位置以及账户、密码等链接信息
-	本地git 仓库：在Config Server文件系统中，每次客户端请求获取配置信息时，Config Server 从git 仓库中clone最新配置到本地，然后在本地git 仓库中读取并返回
-	ServiceA、ServiceB：具体的微服务应用，也就是客户端程序，它指定了Config Server 的地址，从而实现从外部化获取应用自己要用的配置信息。这些应用在启动时，会向Config Server 请求获取配置信息来进行加载。

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fn980xpcglj30eq08edgs.jpg)

&emsp;&emsp;客户端应用从配置管理中获取配置遵循以下执行流程：

1.	应用启动时，根据其配置文件中配置的应用名{application}、环境名{profile}、分支名{label}，向Config Server请求获取配置信息。
2.	Config Server 根据自己维护的git 仓库信息和客户端传递的配置定位信息去查找配置信息
3.	通过 git clone 命令将找到的配置信息下载到 Config Server 的文件系统中
4.	Config Server 创建Spring 的ApplicationContext 实例，并从git 本地仓库中加载配置文件，最后将这些配置内容读取出来返回给客户端使用
5.	客户端应用在获得外部配置文件后加载到客户端的ApplicationContext实例，改配置内容的优先级高于客户端Jar包内部的配置内容，所以在Jar包中重复的内容将不再被加载。

### 个性化配置

-	占位符配置URL：可以使用{application}、{profile}、{label}这些占位符配置git仓库的url地址
-	配置多个仓库：可以用逗号分隔多个{application}/{profile}来配置多个git仓库，当配置多个仓库时，Config Server在启动时会直接克隆第一个仓库的配置库，其他的配置库只有在请求时才会克隆到本地。
-	子目录存储：除了配置git仓库的url地之外，还可以使用searchPaths属性配置子目录，并可使用占位符
-	属性覆盖：通过spring.cloud.config.server.overrides属性来设置键值对参数，这些参数不会被客户端修改，并在客户端获取配置时就会得到这些信息
-	加密解密：可通过encrypt.key 配置秘钥，启动配置中心访问该配置可获得加密后的内容，使用加密内容时在前面加上{cipher}即可
-	服务化配置中心：可把Config Server视为微服务架构中的与其他业务一样的单元，注册到注册中心中，被其他应用所发现来实现配置信息的获取。
-	客户端连接Config Server 失败快速响应与重试：

&emsp;&emsp;客户端会预先加载很多信息，然后才开始连接Config Server，要让客户端优先判断Config Server获取是否正常可在配置文件中配置 spring.cloud.config.failFast=true即可，这样一来在客户端打开后会在控制台打印连接情况。

&emsp;&emsp;当连接失败是由于网络波动等原因引起时，重启的代价比较高，所以Config客户端提供了自动重试的功能，在配置了spring.cloud.config.failFast=true的情况下在pom文件中引入spring-retry和spring-boot-starter-aop依赖即可，不用其他多余配置。

-	动态刷新配置：有时，我们需要对配置内容进行实时更新，这时只需在客户端中新增spring-boot-starter-actuator监控模块的依赖即可
-	本地文件系统：当不使用Git时，也可以使用Config Server本地来作为配置中心的仓库，只需要设置属性 spring.profiles.active=true，然后Config Server就会从应用的resource目录下搜索配置文件，如果需要指定搜索配置文件的路径可通过spring.cloud.config.server.native.searchLocations属性来配置位置。

### 高可用配置

-	传统模式：不需要为服务端做任何额外的配置，只需将所用的Config Server都指向一个git仓库，客户端指定Config Server位置时，只需要配置Config Server上层的负载均衡设备地址即可，如下图

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fn985cmwnmj30fi09jdgv.jpg)

-	服务模式：可把Config Server视为微服务架构中的与其他业务一样的单元，纳入Eureka的服务治理体系中。

----------
<font color="blue">笔者水平有限，若有错漏，欢迎指正，如果转载以及CV操作，请务必注明出处，谢谢！</font>


----------


<font color="red">版权声明：本文为博主原创文章，未经博主允许不得转载。</font>
