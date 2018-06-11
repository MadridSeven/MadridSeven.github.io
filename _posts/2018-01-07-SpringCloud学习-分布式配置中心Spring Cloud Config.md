---
layout: post
title: "SpringCloud学习-分布式配置中心Spring Cloud Config(一)-基础"
date: 2018-01-07
description: "Spring，SpringBoot，SpringCloud"
tag: JAVA进阶学习

---

### 介绍

&emsp;&emsp;先来说说为什么需要 Spring Cloud Config，以及它能用来干什么

- 用途：用来为分布式系统中的基础设施和微服务应用提供集中化的外部配置支持。
- 解决痛点：在分布式系统或微服务应用中，各个模块或服务之间的配置文件都是独立的，但是大多数时候很多配置都是重复的，这样一来造成了很大的维护成本，而Spring Cloud Config 解决了这个问题。

&emsp;&emsp;其次来看看它的特点：

- 分为客户端和服务端两部分。
- 服务端也称为分布式配置中心，用来连接配置仓库并为客户端提供获取配置信息、加密/解密信息等访问接口。
- 客户端则是微服务架构中的各个微服务应用，它们通过指定的配置中心来管理应用的配置内容。
- 默认采用 Git 来存储配置信息。

### 入门实例

&emsp;&emsp;在此，演示一个简单的基于 Git 存储的分布式配置中心，同时对配置的规则进行讲解，
并在客户端中演示如何通过配置指定微服务应用的所属配置中心，并从中获取配置绑定到代码中的过程。

### 服务端

&emsp;&emsp;首先创建一个基础的 SpringBoot 工程，并在 pom.xml 中
引入如下依赖：

```xml
 <parent>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-parent</artifactId>
   <version>1.5.9.RELEASE</version>
   <relativePath/> <!-- lookup parent from repository -->
 </parent>

 <dependencies>
   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter</artifactId>
   </dependency>

   <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-test</artifactId>
     <scope>test</scope>
   </dependency>

   <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-config-server</artifactId>
   </dependency>
 </dependencies>

 <dependencyManagement>
   <dependencies>
     <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-dependencies</artifactId>
       <version>${spring-cloud.version}</version>
       <type>pom</type>
       <scope>import</scope>
     </dependency>
   </dependencies>
 </dependencyManagement>
```

&emsp;&emsp;接着创建 SpringBoot 程序的主类，代码如下：

```Java
//开启Spring Cloud Config的服务端功能
@EnableConfigServer
@SpringBootApplication
public class SpringCloudConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudConfigServerApplication.class, args);
	}
}
```

&emsp;&emsp;在 resources 下的 application.yml 中添加配置：

```yml
server:
  port: 8888
spring:
  application:
    #应用名
    name: config-server
  cloud:
    config:
      server:
        git:
          #配置Git仓库位置
          uri: https://github.com/MadridSeven/learning-spring-boot/
          #配置仓库路径下的相对搜索位置，可配置多个
          search-paths: config_for_spring_cloud/config-repo
          #访问Git仓库使用的用户名和密码
          username: username
          password: password
      label: master
```

&emsp;&emsp;至此一个通过 Spring Cloud Config 实现，并使用 Git 管理配置内容的分布式配置中心
就完成了。
&emsp;&emsp;为了验证上面的努力是正确的，我根据Git配置信息中指定的仓库位置，在 https://github.com/MadridSeven/learning-spring-boot/
下创建了一个 config_for_spring_cloud/config-repo 目录作为配置仓库，并根据不同环境新建了以下四个配置文件：

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fn8inshwhlj307l04lmx3.jpg)

&emsp;&emsp;在这四个配置文件中都设置了一个 from 属性，并对应的设置了不同的值。

&emsp;&emsp;然后我们就可以启动程序直接通过浏览器访问到配置的内容了，在这之前需要交代一下访问配置信息的 URL 与配置文件的映射关系，如下：

- /{application}/{profile}[/{label}]
- /{application}-{profile}.yml
- /{label}/{application}-{profile}.yml
- /{application}-{profile}.properties
- /{label}/{application}-{profile}.properties

&emsp;&emsp;上面的 url 会映射{application}-{profile}.properties 对应的配置文件，{label}可以指定Git上不同的分支，默认为master。

&emsp;&emsp;现在启动之前编写的程序，访问 http://localhost:8888/madridseven/dev，在浏览器中会返回如下信息，其中包含了应用名、环境名等信息。

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fn8j3u5f4xj30pi03ct8q.jpg)

&emsp;&emsp;而控制台中会打印出类似下图的提示：

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fn8j4y1lv3j31ct04f7dy.jpg)

&emsp;&emsp;实质上，程序是通过 git clone 命令将配置内容复制了一份在本地，然后读取并返回给微服务应用加载的。

&emsp;&emsp;该实例代码发布在我的github中，链接：https://github.com/MadridSeven/learning-spring-boot/tree/master/spring-cloud-config/spring-cloud-config-server

### 客户端

&emsp;&emsp;下面我们测试在微服务应用中获取上述配置。

&emsp;&emsp;首先创建一个 SpringBoot 应用，并在pom.xml中引入下述依赖：

```xml
  <parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
```

&emsp;&emsp;创建 SpringBoot 的主类，代码如下：

```Java
@SpringBootApplication
public class SpringCloudConfigClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudConfigClientApplication.class, args);
	}

}
```

&emsp;&emsp;在 resources 下的 bootstrap.yml 中添加配置，来指定获取配置文件的位置等信息，例如：

```yml
spring:
  application:
    #对应配置文件中的{application}部分
    name: madridseven
  cloud:
    config:
      #对应配置文件中的{profile}部分
      profile: dev
      #对应配置文件中的{label}部分
      label: master
      #配置中心地址(之前编写的服务端程序)
      uri: http://localhost:8888/
server:
  port: 8881
```

&emsp;&emsp;创建一个测试的Controller来返回配置中心的属性，通过 @Value("${from}") 绑定配置服务中配置的from属性：

```Java
@RefreshScope
@RestController
public class TestController {

    @Value("${from}")
    private String from;

    @RequestMapping("/from")
    public String from(){
        return this.from;
    }

}
```

&emsp;&emsp;在之前写的主程序中的类名上方配置ComponentScan注解，使其可以扫描到这个类，如下：

```Java
@ComponentScan(basePackages = "springcloud.config.controller")
```

&emsp;&emsp;启动该应用，并访问 http://localhost:8881/from，我们可以获得如下返回内容：git-dev-1.0 。

&emsp;&emsp;该实例代码发布在我的github中，链接：https://github.com/MadridSeven/learning-spring-boot/tree/master/spring-cloud-config/spring-cloud-config-client

&emsp;&emsp;至此，我们已经对 Spring Cloud Config 有了一个大概的了解，并且也使用了其基本的功能，下一篇文章，我将会在这篇的基础上就 Spring Cloud Config 服务端和客户端各自的一些个性化的配置，以及比较高级的应用做一总结。


----------
<font color="blue">笔者水平有限，若有错漏，欢迎指正，如果转载以及CV操作，请务必注明出处，谢谢！</font>


----------


<font color="red">版权声明：本文为博主原创文章，未经博主允许不得转载。</font>
