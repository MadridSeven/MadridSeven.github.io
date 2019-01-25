---
layout: post
title: "SpringCloud微服务基础搭建--传统spring项目如何接入spring cloud体系？"
date: 2019-01-25
description: "Spring，SpringBoot，SpringCloud"
tag: JAVA进阶学习

---

### 项目目录简介

- demo -- spring cloud注册中心
- eureka-client -- spring cloud服务提供者
- eureka-consumer -- spring cloud服务消费者
- mvc-demo -- 传统spring项目接入spring cloud服务体系进行服务的注册和消费

### spring boot的特点
### 核心功能

- 独立运行Spring项目

Spring boot 可以以jar包形式独立运行，运行一个Spring Boot项目只需要通过java -jar xx.jar来运行。（也可以使用war包进行打包并且使用外部tomcat部署，详见https://blog.csdn.net/qq_33512843/article/details/80951741 ）

- 内嵌servlet容器

Spring Boot可以选择内嵌Tomcat、jetty或者Undertow,这样我们无须以war包形式部署项目。

- 提供starter简化Maven配置

spring提供了一系列的start pom来简化Maven的依赖加载，例如，当你使用了spring-boot-starter-web，会自动加入与web开发相关的依赖包。

- 自动装配Spring

SpringBoot会根据在类路径中的jar包，类、为jar包里面的类自动配置Bean，这样会极大地减少我们要使用的配置。当然，SpringBoot只考虑大多数的开发场景，并不是所有的场景，若在实际开发中我们需要配置Bean，而SpringBoot没有提供支持，则可以自定义自动配置。

- 约定大于配置

开发人员只需要关注应用中不符合约定的部分，其它地方采用默认配置

并不是一个新的框架，而是在之前的许多框架的基础上进行了封装，简化了开发过程，使得构建一个spring boot项目变得非常简单。

### Spring Cloud的特点

- Spring Cloud是一个微服务框架，相比Dubbo等RPC框架, Spring Cloud提供的是全套的微服务系统解决方案。
- Spring Cloud对微服务基础框架Netflix的多个开源组件进行了封装，同时又实现了和云端平台以及和Spring Boot开发框架的集成。
- Spring Cloud为微服务架构开发涉及的配置管理，服务治理，熔断机制，智能路由，微代理，控制总线，一次性token，全局一致性锁，leader选举，分布式session，集群状态管理等操作提供了一种简单的开发方式。
- Spring Cloud拥有众多的子项目共同完成微服务架构开发所涉及的方方面面。

### 快速搭建一个简单的Spring Cloud微服务项目
### 构建spring boot工程

快速的构建spring boot工程有两种方式，分别是利用Intellij idea的Spring Initializr进行构建和利用Spring官方提供的快速构建工具来构建，工具链接为： https://start.spring.io/ ，使用Intellij方式构建的方式参考 http://www.spring4all.com/article/247 ，这里讨论使用官方提供的工具来进行安装。

进入工具首页后看到如下页面：

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fziu19or29j30fi06owf5.jpg)

根据需求填写配置后，点击Generate Project下载项目压缩包。

解压后，用IDE以Maven项目导入，导入后项目结构如下图：

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fziu2gpo6jj308z07wmxw.jpg)

此时一个基础的spring boot项目就构建完毕了。

### 创建“服务注册中心”

在刚创建的spring boot工程的基础上在pom中引入如下内容：

```
<dependencies>
   <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
   </dependency>

   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
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
         <version>Finchley.SR2</version>
         <type>pom</type>
         <scope>import</scope>
      </dependency>
   </dependencies>
</dependencyManagement>
```

在程序入口DemoApplication类上添加@EnableEurekaServer注解使其成为一个服务的注册中心：

```
@EnableEurekaServer
@SpringBootApplication
public class DemoApplication {
	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```

在application.properties配置文件中配置以下信息：

```
spring.application.name=eureka-server
eureka.instance.hostname=localhost
server.port=1001
# 禁止注册中心对自身进行注册
eureka.client.register-with-eureka=false
# 注册中心的职责是维护服务实例，故不用去检索服务
eureka.client.fetch-registry=false
```

之后启动工程，即运行程序入口的main方法，访问 http://localhost:1001 ，可以看到如下页面

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fziu327f0dj30fi06lgm3.jpg)

### 创建服务提供者

首先创建一个基础的spring boot工程，命名为eureka-client，在pom文件中加入如下内容：

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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
            <version>Finchley.SR2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

其次，实现/dc请求处理接口，通过DiscoveryClient对象，在日志中打印出服务实例的相关内容。

```
@RestController
public class DcController {
    @Autowired
    DiscoveryClient discoveryClient;
    @GetMapping("/dc")
    public String dc() {
        String services = "Services: " + discoveryClient.getServices();
        System.out.println(services);
        return services;
    }
}
```

最后在应用主类中通过加上@EnableDiscoveryClient注解，该注解能激活Eureka中的DiscoveryClient实现，这样才能实现Controller中对服务信息的输出。

```
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class, args);
    }
}
```

我们在完成了服务内容的实现之后，再继续对application.properties做一些配置工作，具体如下：

```
#指定微服务的名称后续在调用的时候只需要使用该名称就可以进行服务的访问
spring.application.name=eureka-client
server.port=2001
#对应服务注册中心的配置内容，指定服务注册中心的位置
eureka.client.serviceUrl.defaultZone=http://localhost:1001/eureka/
```

之后运行该工程，再次访问http://localhost:1001 ,可以看到如下图的内容，说明我们的服务已经被注册到注册中心了。

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fziu3nih6qj30fi017t8n.jpg)

我们也可以通过直接访问eureka-client服务提供的/dc接口来获取当前的服务清单，只需要访问http://localhost:2001/dc ,可以得到输出Services: [eureka-client]，方括号中的内容就是接口返回的服务清单。

至此一个简单的服务提供者就实现好了。

### 创建服务消费者：

在Spring Cloud中提供了大量的与服务治理相关的抽象接口，包括DiscoveryClient、LoadBalancerClient等。

LoadBalancerClient是一个负载均衡客户端的抽象定义，下面就使用该接口实现服务的消费。

先创建一个基础的spring boot工程，命名为eureka-consumer，在pom文件中加入如下内容：

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
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
            <version>Finchley.SR2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

配置application.properties，指定注册中心的地址：

```
spring.application.name=eureka-consumer
server.port=2101
eureka.client.serviceUrl.defaultZone=http://localhost:1001/eureka/
```

在启动类中初始化RestTemplate，用来真正发起REST请求。@EnableDiscoveryClient注解用来将当前应用加入到服务治理体系中。

```
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaConsumerApplication {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    public static void main(String[] args) {
        SpringApplication.run(EurekaConsumerApplication.class, args);
    }
}
```

创建一个接口用来消费eureka-client提供的接口：

```
@RestController
public class DcController {
    @Autowired
    LoadBalancerClient loadBalancerClient;
    @Autowired
    RestTemplate restTemplate;
    @GetMapping("/consumer")
    public String dc() {
        //通过loadBalancerClient的choose函数来负载均衡的选出一个eureka-client的服务实例
        ServiceInstance serviceInstance = loadBalancerClient.choose("eureka-client");
        //获取真实的服务地址
        String url = "http://" + serviceInstance.getHost() + ":" + serviceInstance.getPort() + "/dc";
        System.out.println(url);
        //对服务进行调用
        return restTemplate.getForObject(url, String.class);
    }
}
```

之后启动该项目，先访问注册中心，http://localhost:1001 发现消费者已经被注册到注册中心中去了，如图

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fziu45atb8j30fi01iaa2.jpg)

访问http://localhost:2101/consumer 对服务进行调用，返回Services: [eureka-client, eureka-consumer]

此时，一个服务的注册、发现、消费的整个过程就都已经实现了。

### 传统spring项目（非spring boot）进行服务的注册和消费

当使用传统spring体系开发的项目想要接入spring cloud注册中心并进行服务的注册和发现时，可参考如下内容：

### 添加pom依赖

```
<!-- eureka原生的服务发现体系 -->
<dependency>
  <groupId>com.netflix.eureka</groupId>
  <artifactId>eureka-client</artifactId>
  <version>1.4.12</version>
</dependency>
<dependency>
  <groupId>com.netflix.archaius</groupId>
  <artifactId>archaius-core</artifactId>
  <version>0.7.3</version>
</dependency>
```

### 配置文件

之后新建一个配置文件config.properties，也可以命名为其他，该文件会被spring进行扫描。其中包含如下内容：

```
eureka.region=default
#服务的名字
eureka.name=mvc-consumer
eureka.preferSameZone=true
eureka.shouldUseDns=false
eureka.us-east-1.availabilityZones=default
#配置注册中心的地址
eureka.serviceUrl.default=http://localhost:1001/eureka/
```

### 配置监听器

新建一个监听器EurekaInitAndRegisterListener将应用注册到注册中心中，如下

```
public class EurekaInitAndRegisterListener implements ServletContextListener {

    private static final DynamicPropertyFactory configInstance = DynamicPropertyFactory
            .getInstance();
    public void contextInitialized(ServletContextEvent sce) {
        /*设置被读取配置文件名称  默认config.properties*/
        System.setProperty("archaius.configurationSource.defaultFileName", "config.properties");
        /*注册*/
        registerWithEureka();
    }
    public void registerWithEureka() {
        /*加载本地配置文件 根据配置初始化这台 Eureka Application Service 并且注册到 Eureka Server*/
        DiscoveryManager.getInstance().initComponent(
	     /*MyInstanceConfig的代码后面有说到*/
                new MyInstanceConfig(),
                new DefaultEurekaClientConfig());

        /**本台 Application Service 已启动，准备好侍服网络请求*/
        ApplicationInfoManager.getInstance().setInstanceStatus(
                InstanceInfo.InstanceStatus.UP);
    }

    public void contextDestroyed(ServletContextEvent sce) {
        DiscoveryManager.getInstance().shutdownComponent();
    }
}
```

MyInstanceConfig类的代码如下：

```
public class MyInstanceConfig extends MyDataCenterInstanceConfig {

    /**
     * 优先使用ip 替换 hostname
     */
    @Override
    public String getHostName(boolean refresh) {
        try {
            return InetAddress.getLocalHost().getHostAddress();
        } catch (UnknownHostException e) {
            return super.getHostName(refresh);
        }
    }

    /***
     *获取本机启动中tomcat端口号
     */
    @Override
    public int getNonSecurePort(){
        int tomcatPort;
        try {
            MBeanServer beanServer = ManagementFactory.getPlatformMBeanServer();
            Set<ObjectName> objectNames = beanServer.queryNames(new ObjectName("*:type=Connector,*"),
                    Query.match(Query.attr("protocol"), Query.value("HTTP/1.1")));

            tomcatPort = Integer.valueOf(objectNames.iterator().next().getKeyProperty("port"));
        }catch (Exception e){
            return super.getNonSecurePort();
        }
        return tomcatPort;
    }
}
```

在web.xml中配置该监听器：

```
<listener>
    <listener-class>com.mvc.demo.EurekaInitAndRegisterListener</listener-class>
</listener>
```

此时启动项目就会发现服务已经被注册到注册中心中并被管理起来了（如图）

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fziu4q05r2j30fi01gq2x.jpg)

### 请求服务接口

服务被注册到注册中心之后，就可以通过RestTemplate请求其他服务的接口了，下面的示例代码提供了一个接口，该接口会请求tmop-server服务的一个接口，并返回其返回值

```
@RequestMapping(value = "/consumer",produces = "application/json;charset=UTF-8")
public String dc(HttpServletRequest request){
    RestTemplate restTemplate = new RestTemplate();
    //获取服务的真实地址
    String myServer = this.getRealServerHost("eureka-client");
    String url = "http://" + myServer + "/dc";
    //请求头
    HttpHeaders headers = new HttpHeaders();
    Enumeration<String> headerNames = request.getHeaderNames();
    while (headerNames.hasMoreElements()) {
        String key = (String) headerNames.nextElement();
        String value = request.getHeader(key);
        headers.add(key, value);
    }
    HttpEntity entity = new HttpEntity(headers);
    //使用restTemplate发起请求并返回结果
    ResponseEntity responseEntity = restTemplate.exchange(url, HttpMethod.GET, entity, String.class);
    return (String) responseEntity.getBody();
}
```

getRealServerHost方法如下：

```
public String getRealServerHost(String serviceId){
    InstanceInfo serverInfo = DiscoveryManager.getInstance()
                    .getDiscoveryClient()
                    .getNextServerFromEureka(serviceId, false);
    String realServerName = serverInfo.getIPAddr() + ":" + serverInfo.getPort();
    return realServerName;
}
```

启动项目，访问该接口，返回如下信息：Services: [eureka-client, eureka-consumer, mvc-consumer]。

至此，一个传统的Spring工程就被纳入到了spring cloud体系中，并可以进行服务的注册和消费了。

### 一些注意事项

- 在使用war包部署spring boot项目时，一定要在pom中排除spring boot内置的tomcat容器，不然会有jar包冲突，使用war包部署的方式详见：
https://blog.csdn.net/qq_33512843/article/details/80951741
- 静态资源处理：spring boot默认会依次从resources目录下的public、resources、static目录下查找静态文件，详见：
https://www.cnblogs.com/magicalSam/p/7189476.html
- 当使用war包部署，并且一个tomcat容器中要部署多个spring boot项目时，应该在配置文件中添加如下配置：
```
#唯一名称标记，可为任何名称，项目和项目间不能重复
spring.jmx.default-domain=eureka-server
```
- 当消费服务时，服务的消费者一定要知道所消费的服务所返回的结果类型
