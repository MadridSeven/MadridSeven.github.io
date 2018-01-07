---
layout: post
title: "SpringCloud学习-SpringBoot入门及使用Mysql"
date: 2018-01-05
description: "Spring，SpringBoot，SpringCloud"
tag: JAVA进阶学习

---

### 为什么要使用SpringBoot

&emsp;&emsp;Spring Boot可以说是至少近5年来Spring乃至整个Java社区最有影响力
的项目之一，也被人看作是：Java EE开发的颠覆者。

&emsp;&emsp;


### SpringBoot的优点

- 去除了之前Spring项目的大量的xml配置文件
- 简化了复杂的依赖管理
- 配合各种starter使用，基本上可以做到自动化配置
- 快速启动容器
- 嵌入式Tomcat，Jetty容器，配合Maven或Gradle等构件工具打成Jar包后，Java -jar 进行部署运行还是蛮简单的



### 入门HelloWorld


&emsp;&emsp;要使用 SpringBoot 框架，先要安装 git 和 maven，
这两个工具的安装过程就不赘述了，网上很多；另外本文所使用的 ide 是 IntelliJ IDEA;使用的jdk版本为1.8。

&emsp;&emsp;要创建一个基础的 SpringBoot 项目是一件很简单的事情，基本常用的创建方法有 Maven 和 Spring Initializr
两种，这里先介绍前者，后者会在后续的文章中介绍。至于如何使用 idea 创建一个 Maven 项目也就不多介绍了。

&emsp;&emsp;首先新建一个 Maven 项目引入如下依赖：

```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.9.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
```

&emsp;&emsp;这里只引入了一个依赖配置 spring-boot-starter-web
和一个 parent 配置 spring-boot-starter-parent，但是只引入这
两个依赖配置 SpringBoot 就会导入整个 springframework 依赖，
以及 logging、slf4j、jackson、tomcat 插件等，所有这些都是一个
 Web 项目可能需要用到的东西，可以说是特别方便了。

&emsp;&emsp;之后在主目录下创建一个类，用作 SpringBoot 的程序入口，代码如下：

```Java
/**
 * SpringBootApplication注解表示其是一个Spring Boot应用
 * RestController注解标注这个程序是一个控制器
 */
@SpringBootApplication
@RestController
public class Application {
    @RequestMapping("/")
    String home(){
        return "hello";
    }
    public static void main(String[] args) {
        SpringApplication.run(Application.class,args);
    }
}

```

&emsp;&emsp;虽然仅仅使用了几行代码，但其实这已经是一个完整的 Web 项目了。

&emsp;&emsp;之后在工程的 resource 文件夹中创建一个 application.yml 文件，
这个文件会被发布在 classpath 中并被 SpringBoot 自动读取。代码如下：

```yml
server:
  port: 8080
  tomcat:
    uri-encoding: UTF-8
```

&emsp;&emsp;之后运行之前写好的主程序，然后在浏览器的地址栏中输入 http://localhost:8080/。这样就
可以看到我们期望的输出字符：hello。

&emsp;&emsp;至此一个最基本的 SpringBoot 应用就开发完成了。

&emsp;&emsp;该实例的代码发布在我的github中，链接：https://github.com/MadridSeven/learning-spring-boot/tree/master/spring-boot-hello

&emsp;&emsp;

### SpringBoot 使用Mysql

&emsp;&emsp; 为了使用 JPA 和 Mysql，首先在工程中引入它们的 Maven 依赖，代码如下：

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
```

&emsp;&emsp;其余的配置与上一个基础应用一样。

### 实体建模

&emsp;&emsp;这个实例，我们考虑建三张数据表，部门、用户、角色表，其中
部门和用户是一对多关系，用户和角色是多对多关系。

&emsp;&emsp;部门实体类代码如下所示，这里省略了Geeter/Setter方法：

```Java
@Entity
@Table(name = "department")
public class Department {

    //指定其为主键并且自增
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    public Department() {

    }

    ......
}
```

&emsp;&emsp;用户实体类代码如下，省略了Geeter/Setter方法和toString方法：

```Java
@Entity
@Table(name = "user")
public class User implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm")
    private Date createDate;

    public User(){

    }

    //部门和用户为一对多关系，在部门("一")的实体类中设置ManyToOne关联
    @ManyToOne
    //在生成的数据表中使用did表示部门的id
    @JoinColumn(name="did")
    //防止关系对象递归访问
    @JsonBackReference
    private Department department;

    //用户和角色为多对多关系
    @ManyToMany(cascade = {},fetch = FetchType.EAGER)
    //多对多需要使用中间表
    @JoinTable(name = "user_role",joinColumns = {@JoinColumn(name = "user_id")},inverseJoinColumns = {@JoinColumn(name = "roles_id")})
    private List<Role> roles;
    ......
}
```

&emsp;&emsp;角色实体建模：

```Java
@Entity
@Table(name = "role")
public class Role implements Serializable{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    public Role() {
    }
    ......
}
```

&emsp;&emsp;

### 实体持久化

&emsp;&emsp;通过建立上面的三个实体类，我们已经实现了使用 Java 的普通对象(POJO)
与数据库表建立映射关系(ORM)，接下来使用JPA实现持久化。

&emsp;&emsp;用户实体持久化的代码如下所示，它是一个接口，继承了JPA资源库 JpaRepository 接口。

```Java
//Repository注解说明该接口是一个资源库
@Repository
public interface UserRepository extends JpaRepository<User,Long> {
}
```

&emsp;&emsp;使用同样的方法，可以定义部门实体和角色实体的资源库接口，只要注意使用的参数是各自的实体对象即可。

&emsp;&emsp;这样就实现了存取数据库的功能了，我们现在就可以对数据库进行增删改查、分页查询等操作了。
但是用个问题，我们定义的实体资源库接口并没有声明一个方法，也没有一个实现类，为什么就可以进行数据库操作呢？
其实我们看看 JpaRepository 的继承关系就可以了解个大概了。

![](http://ww1.sinaimg.cn/large/006CsMmSgy1fn7wlijc2mj30ek0dwjri.jpg)

&emsp;&emsp;其中 JpaRepository 继承于 PagingAndSortingRepository，它提供了分页和排序功能，
PagingAndSortingRepository 继承于 CrudRepository，它提供了简单的增删改查功能。

&emsp;&emsp;JPA 还提供了一些自定义生命方法的规则，例如，在接口中使用关键字 findBy、readBy、getBy作为方法名的
前缀，拼接实体类中的属性字段，并可选择凭借一些SQL查询关键字来组成一个查询方法。

&emsp;&emsp;

### Mysql 测试

&emsp;&emsp;增加一个JPA的配置类，代码如下所示。

```Java
@Configuration
@EnableJpaRepositories(basePackages = "byh.**.repository")
@Order(Ordered.HIGHEST_PRECEDENCE)
//启用Jpa事务管理
@EnableTransactionManagement(proxyTargetClass = true)
//启用Jpa资源库，指定接口资源库位置
@EntityScan(basePackages = "byh.**.entity")
public class JpaConfiguration {

    @Bean
    PersistenceExceptionTranslationPostProcessor persistenceExceptionTranslationPostProcessor(){
        return new PersistenceExceptionTranslationPostProcessor();
    }

}
```

&emsp;&emsp;其次在Mysql数据库服务器中创建一个数据库test，不用创建数据库的表结构，程序运行时将会按照
实体的定义自动创建。

&emsp;&emsp;然后，在配置文件 application.yml 中使用如下配置：

```yml
spring:
  profiles:
    active: prod
  datasource:
    url: jdbc:mysql://localhost:3306/test?characterEncoding=utf8
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver
  jpa:
    database: mysql
    show-sql: true
    hibernate:
      ddl-auto: update
      naming:
        strategy: org.hibernate.cfg.ImprovedNamingStrategy
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL5Dialect
server:
  port: 8080
  tomcat:
    uri-encoding: UTF-8
```

&emsp;&emsp;写一个主程序来测试，代码如下：

```Java
public class MysqlTest {

    @Autowired
    UserRepository userRepository;
    @Autowired
    DepartmentRepository departmentRepository;
    @Autowired
    RoleRepository roleRepository;

    @RequestMapping("/find")
    public String findPage(){
        this.initData();
        Pageable pageable = new PageRequest(0,10,new Sort(Sort.Direction.ASC,"id"));
        Page<User> page = userRepository.findAll(pageable);
        String result = "";
        for(User user: page.getContent()){
            result += user.toString();
        }
        return result;
    }

    public void initData(){
        userRepository.deleteAll();
        roleRepository.deleteAll();
        departmentRepository.deleteAll();

        Department department = new Department();
        department.setName("开发部");
        departmentRepository.save(department);

        Role role = new Role();
        role.setName("admin");
        roleRepository.save(role);

        User user = new User();
        user.setName("user");
        user.setCreateDate(new Date());
        user.setDepartment(department);
        List<Role> roles = roleRepository.findAll();
        user.setRoles(roles);
        userRepository.save(user);
    }

    public static void main(String[] args) {
        SpringApplication.run(MysqlTest.class,args);
    }

}
```

&emsp;&emsp;运行该程序，在浏览器中输入 http://localhost:8080/find，之后会看到从数据库中查询到的值。

&emsp;&emsp;该实例的代码发布在我的github中，链接：https://github.com/MadridSeven/learning-spring-boot/tree/master/spring-boot-dataBase



----------
<font color="blue">笔者水平有限，若有错漏，欢迎指正，如果转载以及CV操作，请务必注明出处，谢谢！</font>


----------


<font color="red">版权声明：本文为博主原创文章，未经博主允许不得转载。</font>
