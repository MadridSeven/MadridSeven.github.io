---
layout: post
title: "Spring-AOP切面编程总结"
date: 2017-03-20 
description: "JAVA，Spring，AOP，切面编程"
tag: 框架 
---
写在前面：

&emsp;&emsp;之前写了三篇JAVA基础进阶、一篇JAVA源码解析，今天又过来写框架，大家别担心，另外两个以后还会继续写的，给大家预告下，下一篇博客会写JAVA基础进阶的《JAVA反射机制》，然后会写JAVA源码解析的集合源码解析，那块应该一两个集合类型就是一篇博客了吧。近期打算把博客搬到github上面去了，但是这里以后还会继续更新的，两边都一样。





# .1 AOP简介

&emsp;&emsp;AOP是面向切面编程(Aspect-Oriented Programming)的缩写，是对传统的面向对象编程的一种补充，下面来介绍下在AOP中经常用到的一些术语。

> - <font color="red">切面(Aspect):横切关注点(跨越应用程序多个模块的功能)被模块化的特殊对象</font>
 
 >- <font color="red">通知(Advice):切面必须要完成的工作</font>
 
 >- <font color="red">目标(Target):被通知的对象</font>
 
 >- <font color="red">代理(Proxy): 向目标对象应用通知之后创建的对象</font>
 
 >- <font color="red">连接点（Joinpoint）：程序执行的某个特定位置：如类某个方法调用前、调用后、方法抛出异常后等。</font>
 
 >- <font color="red">切点(pointcut):  一个类中可以有多个连接点  ,就和每个类中可以有多个方法一样，AOP通过切点定位到特定的连接点。  可以这么理解  ：连接点相当于数据库中的记录，切点则相当于查询条件，切点使用类和方法作为连接点的查询条件。</font>
 
&emsp;&emsp;AOP切面编程是围绕着通知注解这个东西展开的，在SpringAOP中共有五种类型的通知注解：

>-   @Before: 前置通知，在方法执行之前执行

>- @After: 后置通知，在方法执行之后执行

>- @AfterRunning:返回通知，在方法返回结果之后执行

>- @AfterThrowing:异常通知，在方法抛出异常之后

>- @Around:环绕通知，围绕着方法执行  （并不常用）



# .2 一个小需求

&emsp;&emsp; 我通过一个小小的需求来向大家展示SpringAOP编程的各个细节，比如说，我们写了两个方法：一个方法将两个整形的数字做加法运算、一个方法将两个整形数字做除法运算，这样一个需求看起来非常简单。但是，我们要求， **在方法开始时、结束时、返回时、发生异常时，分别在控制台上输出一段文字来说明当前的状态**  ，这时该怎么办呢？？

&emsp;&emsp;  我们通过AOP可以很轻松的解决这个问题。

# .3 通过注解的方式配置AOP

&emsp;&emsp; 首先建立一个JAVA项目，在项目下导入Spring的开发包，然后创建一个接口ArithmeticCalculator，在接口中定义两个方法，add()和div()。代码如下：

```java
package com.byh.aop.hello;

public interface ArithmeticCalculator {
	
	int add(int i,int j);
	int div(int i,int j);

}
```

 &emsp;&emsp; 在src下创建Spring的配置文件applicationContext.xml，并<font color="red">在Namespaces选项中勾选aop和context这两个选项</font>，之后在applicationContext.xml中启用 AspectJ 注解支持，并指定IOC容器的扫描范围。代码如下：

```xml
	<context:component-scan base-package="com.byh.aop.hello"></context:component-scan>  <!-- 指定IOC容器的扫描范围 -->
	<aop:aspectj-autoproxy></aop:aspectj-autoproxy> <!-- 启用 AspectJ 注解支持 -->
```

&emsp;&emsp; 添加该接口的实现类ArithmeticCalculatorImpl，实现ArithmeticCalculator接口的两个方法。代码如下：

```java
package com.byh.aop.hello;
import org.springframework.stereotype.Component;

@Component //将该类注入到IOC容器中
public class ArithmeticCalculatorImpl implements ArithmeticCalculator  {

	@Override
	public int add(int i, int j) {

		int result = i+j;
	
		return result;
	}

 

	@Override
	public int div(int i, int j) {
	
		int result = i/j;
		
		return result;
	}

}
```
&emsp;&emsp; 写一个测试类来测试，代码如下：

```java
package com.byh.aop.hello;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Main {

	public static void main(String[] args) {
		
		ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
		ArithmeticCalculator arithmeticCalculator = ctx.getBean(ArithmeticCalculator.class);
		
		int result = arithmeticCalculator.add(3, 6);
		System.out.println("result:"+result);
		
		result = arithmeticCalculator.div(12, 6);
		System.out.println("result:"+result);
	}
	
}
```

&emsp;&emsp; 运行之后我们发现运行是成功的，但是这还没有达到我们的要求，在需求中我们要求在方法开始时、结束时、返回时、发生异常时，分别在控制台上输出一段文字来说明当前的状态，现在我们就用AOP来实现。

## .3-1 前置通知
&emsp;&emsp; 首先我们来想如何让方法开始时在控制台输出一段话呢？这是就要用到AOP的前置通知了。前置通知就是在方法执行之前执行的通知，前置通知使用@Before注解，并将切入点表达式的值也就是该方法的路径，作为其注解值。<font color="red">如果要把一个类声明为一个切面的话，只要在该类的前面加上@Aspect注解就可以了。可以在通知方法中声明一个类型为 JoinPoint 的参数.。然后就能访问链接细节. 如方法名称和参数值。</font>

我们来新建一个类名为LoggingAspect，并在该类中实现前置通知，代码如下：

```java
@Component //将该类注入到IOC容器 
@Aspect  //将该类声明为切面
public class LoggingAspect {
		
	@Before(value = "execution(public int com.byh.aop.hello.ArithmeticCalculator.*(..))")//将该方法声明为前置通知，切入点表达式的值为连接点的路径
	public void beforeMethod(JoinPoint joinPoint){
		String methodName = joinPoint.getSignature().getName();
		List<Object> args = Arrays.asList(joinPoint.getArgs());
		System.out.println("前置通知：   The method "+methodName+" begins with "+args);
	}
		
}
```

## .3-2 后置通知

&emsp;&emsp;后置通知是在连接点完成之后执行的, 即连接点返回结果或者抛出异常的时候,。<font color=red >一个切面可以包括一个或者多个通知</font>，所以根据我们的需求，我们在刚才前置通知的基础上在LoggingAspect类中添加实现后置通知的方法即可，代码如下：

```java
@After(value = "execution(public int com.byh.aop.hello.ArithmeticCalculator.*(..))")
	public void afterMethod(JoinPoint joinPoint){
		
		String methodName = joinPoint.getSignature().getName();
		System.out.println("后置通知：   end method "+methodName);
		
	}
```

&emsp;&emsp;这时不知道大家有没有发现一个问题，我们在每一个通知的前面都要写上切点表达式，而且切点表达式的内容是一样的，这样就不符合我们编码的风格了，该如何进行简化呢？我们可以声明一个类去指定切入点表达式的值，这个方法中什么都不用写，只需要在方法前使用@Pointcut注解来指定切点表达式即可，代码如下。

```java
@Pointcut("execution(public int com.byh.aop.hello.ArithmeticCalculator.*(..))")
	public void declareJointPointExpression(){}
```

&emsp;&emsp;将该方法加入到LoggingAspect类中，并且写在所有通知之前，这样一来，当需要生命切入点表达式时，直接将该方法的名字写上就好了，我们修改之前写的前置通知和后置通知的切入点表达式声明方法，代码如下：

```java
@Before("declareJointPointExpression()")//修改切入点表达式的声明方法
	public void beforeMethod(JoinPoint joinPoint){
		...
	}
```

## .3-3 返回通知

&emsp;&emsp;<font color="red">无论连接点是正常返回还是抛出异常, 后置通知都会执行</font>。如果想在连接点返回的时候记录日志, 应使用返回通知。代码如下：

```java
@AfterReturning(value="declareJointPointExpression()",
			returning="result")//returning指定返回的参数
	public void afterReturning(JoinPoint joinPoint,Object result){
		String methodName = joinPoint.getSignature().getName();
		System.out.println("返回通知：   The method "+methodName+" ends with "+result);
	}
```

## .3-4 异常通知


&emsp;&emsp;异常通知只在连接点抛出异常时才会执行，将 throwing 属性添加到 @AfterThrowing 注解中, 也可以访问连接点抛出的异常..Throwable 是所有错误和异常类的超类.。所以在异常通知方法可以捕获到任何错误和异常。如果只对某种特殊的异常类型感兴趣, 可以将参数声明为其他异常的参数类型， 然后通知就只在抛出这个类型及其子类的异常时才被执行。代码如下：

```java
@AfterThrowing(value="declareJointPointExpression()",
			throwing="ex")
	public void afterThrowing(JoinPoint joinPoint,Exception ex){
		
		String methodName = joinPoint.getSignature().getName();
		System.out.println("异常通知：   The method "+methodName+" occurs exception: "+ex);
		
	}
```

&emsp;&emsp;这时我们再去运行下测试方法，发现运行的结果已经变成了这样：

	前置通知：   The method add begins with [3, 6]
	后置通知：   end method add
	返回通知：   The method add ends with 9
	result:9
	前置通知：   The method div begins with [12, 6]
	后置通知：   end method div
	返回通知：   The method div ends with 2
	result:2

&emsp;&emsp;这就表明我们写的几个通知已经起到它该有的作用了，测试异常通知的话，只需将除法的被除数改为0即可，这里就不做演示了，至此我们已经了解AOP到底是如何使用的了。

# .4 通过基于 XML 的配置来配置AOP

&emsp;&emsp;除了使用注解的方式去配置之外，我们还可以使用配置文件的方式去配置。

&emsp;&emsp;在 Bean 配置文件中, 所有的 Spring AOP 配置都必须定义在 aop:config 元素内部。

&emsp;&emsp;对于每个切面而言, 都要创建一个 aop:aspect 元素来为具体的切面实现引用后端 Bean 实例。

&emsp;&emsp;切入点使用 aop:pointcut 元素声明，切入点必须定义在 aop:aspect 元素下, 或者直接定义在 aop:config 元素下。区别是定义在 aop:aspect 元素下: 只对当前切面有效，定义在 aop:config 元素下: 对所有切面都有效。

&emsp;&emsp;修改applicationContext.xml，代码如下：

```xml
<context:component-scan base-package="com.byh.aop.hello"></context:component-scan>

	<!-- 配置AOP -->
	
<aop:config>
	<!-- 配置切点表达式 -->
	<aop:pointcut expression="execution(* com.byh.aop.hello.ArithmeticCalculator.*(..))" id="pointcut"/>
	<!--声明切面-->
	<aop:aspect ref="loggingAspect">
		<!--前置通知-->
		<aop:before method="beforeMethod" pointcut-ref="pointcut"/>
		<!--后置通知-->
		<aop:after method="afterMethod" pointcut-ref="pointcut"/>
		<!--返回通知-->
		<aop:after-returning method="afterReturning" pointcut-ref="pointcut" returning="result"/>
		<!--异常通知-->
		<aop:after-throwing method="afterThrowing" pointcut-ref="pointcut" throwing="ex"/>
	</aop:aspect>
		
	
</aop:config>
```

&emsp;&emsp;然后我们将LoggingAspect类中所有关于AOP的注解都删除掉，删除之后代码如下：

```java
@Component
public class LoggingAspect {
	
	public void beforeMethod(JoinPoint joinPoint){
		String methodName = joinPoint.getSignature().getName();
		List<Object> args = Arrays.asList(joinPoint.getArgs());
		System.out.println("前置通知：   The method "+methodName+" begins with "+args);
	}
	
	public void afterMethod(JoinPoint joinPoint){
		
		String methodName = joinPoint.getSignature().getName();
		System.out.println("后置通知：   end method "+methodName);
		
	}

	public void afterReturning(JoinPoint joinPoint,Object result){
		String methodName = joinPoint.getSignature().getName();
		System.out.println("返回通知：   The method "+methodName+" ends with "+result);
	}

	public void afterThrowing(JoinPoint joinPoint,Exception ex){
		
		String methodName = joinPoint.getSignature().getName();
		System.out.println("异常通知：   The method "+methodName+" occurs exception: "+ex);
		
	}
	
}
```

&emsp;&emsp;之后运行测试方法，发现运行结果与使用注解进行配置的方式是一样的。


----------


<font color="blue">笔者水平有限，若有错漏，欢迎指正，如果转载以及CV操作，请务必注明出处，谢谢！</font>


----------


<font color="red">版权声明：本文为博主原创文章，未经博主允许不得转载。</font>