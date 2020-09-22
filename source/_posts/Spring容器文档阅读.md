---
title: Spring容器文档阅读
date: 2020-09-23 00:55:49
tags:
    - Spring
    - 文档
categories:
    - Spring
---
# Spring容器文档阅读

容器代码位于 `org.springframework.beans` 和 `org.springframework.context`包下面



---

## 容器的基本的接口

基本接口：BeanFactory，ApplicationContext 是BeanFactory的子接口，提供了企业级应用的功能，拥有更加高级的功能

平常的业务开发基本上都是层ApplicationContext的的实现类开始，因为其提供了功能强大的实现

常用的容器实现类：

Web类：
- XmlWebApplicationContext
- AnnotationConfigWebApplicationContext
- GroovyWebApplicationContext

非Web类：

- ClassPathXmlApplicationContext
- GenericXmlApplicationContext
- AnnotationConfigApplicationContext
- GenericGroovyApplicationContext
- FileSystemXmlApplicationContext



---

## BeanDefinition和BeanDefinitionReader

### BeanDefinition

BeanDefinition用于存放Bean的定义元信息

Spring容器的的MetaData定义和具体的元信息的定义形式无关，是完全解耦的。支持XML，注解，Java配置类都可以用于定于Spring容器都元信息。Spring2.5开始支持注解，Spring3.0开始支持基于Java配置类的元信息定义

容器中记录了BeanDefinition用于定义Bean的定义和其它一些信息，包括：

- 包限定的类
- Bean的名字
- 作用域
- 构造器参数
- Bean行为定义，包含Scope，lifeCycle callbacls等
- 依赖的其它Bean
- 其它一些设置信息



![](https://img2020.cnblogs.com/blog/1513538/202009/1513538-20200923003605168-104158597.png)


常见的实现类：

- AnnotatedGenericBeanDefinition：可以读取注解来定义元信息的BeanDefinition
- RootBeanDefinition：能够单独作为一个BeanDefinition，或者是别的BeanDefinition的父定义，但不能作为一个BeanDefinition的子定义
- ChildBeanDefinition：不能单独作为一个BeanDefinition，必须要依赖一个父BeanDefinition
- ConfigurationClassBeanDefinition：是一个为Java的配置类生成的过渡性BeanDefinition，用于确定一个Bean是否需要覆盖另一个Bean的情况
- ScannedGenericBeanDefinition：和AnnotatedGenericBeanDefinition功能差不多，但是增加了扫描.Class的特性



### BeanDefinitionReader

BeanDefinitionReader用于读取不同种类的容器定义形式的元数据的接口

有实现类：
- GroovyBeanDefinitionReader：通过Groovy的的方式定义元信息
- PropertiesBeanDefinitionReader：通过Property格式定义元数据
- XmlBeanDefinitionReader：常用的通过XML的格式定义元数据



---

## Bean的生命周期

1. 初始化容器
2. 读取Bean的定义数据，生成`BeanDifinition`并缓存到BeanFactory中
3. 完成初始化容器，回调`BeanFactoryPostProcessor`接口的实现类的`postProcessBeanFactory`
4. 容器开始遍历缓存的`BeanDifinition`集合并实例化非懒加载的Bean（如果是懒加载，则在主动向容器获取的时候开始Bean生命周期）
   1. 实例化Bean，如果是构造器注入依赖则使用指定的构造器执行实例化操作
   2. 设置对象属性，通过反射调用` setXXX()`方法将依赖的Bean注入
   3. 检查当前的Bean是否实现了各种，*Aware相关接口。如`ApplicationContextAware`，`BeanNameAware`，`BeanClassLoaderAware`等。如果实现就调用相关方法
   4. 将当前Bean的实例作为参数调用一系列的`BeanPostProcessor`接口的实现类的`postProcessBeforeInitialization`前置处理方法
   5. 检查当前Bean的是否实现了`InitializingBean`接口，如果实现了就调用`afterPropertiesSet`方法
   6. 检查是否有自定义的`init-method`或者Bean中有被`@PostConstruct`注解的方法，有就调用
   7. 将当前Bean的实例作为参数调用一系列的`BeanPostProcessor`接口的实现类的`postProcessAfterInitialization`后置处理方法
   8. 注册必要的Destruction相关回调接口
   9. 正常使用中
   10. 当Application关闭的时候，检查Bean是否实现了`DisposableBean`接口，有就调用`destroy`方法
   11. 检查Bean是否有配置自定义的 `destroy-method`方法或者`@PreDestroy`注解的方法，有就调用。另外Spring的容器会自动调用实现了`java.lang.AutoCloseable` 或者`java.io.Closeable`的Bean的接口
5. Application关闭成功



## 容器与Bean的生命周期回调扩展点

有以下3类扩展方法，用于不同的生命周期节点和不同的作用域

### 1. Bean级别扩展点：

#### 接口：

- 普通接口：InitializingBean，DisposableBean
- 各种Aware接口：ApplicationContextAware, BeanNameAware, BeanClassLoaderAware等
- 特殊的接口：Lifecycle，LifecycleProcessor

其它的一些Aware类型接口都可以用于Bean的实现


![](https://img2020.cnblogs.com/blog/1513538/202009/1513538-20200923003636984-63802592.png)


#### 注解：@PostConstruct，@PreDestroy



### 2. 容器级别扩展点：

#### 接口：BeanPostProcessor，Ordered，BeanFactoryPostProcessor

##### BeanPostProcessor

BeanPostProcessor接口用于在Spring容器初始化实例后的一些回调方法，可用于自定义Bean初始化业务逻辑。

Spring-Aop的应用：
在AOP编程中，要实现AOP有两种方式，如果是按接口注入的方式，那么可以使用JDK的动态代理生成代理类。如果按类注入的方式，就需要CGLIB来生成字节码注入的方式来生成代理类。而上面两种方式的共同点都是要向使用者隐瞒真实的类型信息，让使用者认为代理类就是被代理类本身。这个过程就可以通过BeanPostProcessor的回调函数进行操作，生成代理类后直接返回该代理类到容器中，容器会认为该代理类是代理类本身。在注入的时候，就会把代理类注入。



##### BeanFactoryPostProcessor

BeanFactoryPostProcessor接口用于在容器初始化并加载完所有BeanDefinition后初始化实例之前。在这里我们可以自定义的去配置Bean的元信息，也就是BeanDefinition。

##### Ordered

Ordered接口是用于排序BeanPostProcessor和BeanFactoryPostProcessor的实现类的。如果存在多个BeanPostProcessor和BeanFactoryPostProcessor。那么，Spring会使用Ordered接口来给多个处理器进行排序，按从小到大的顺序进行处理




### 3. 工厂Bean：

一类特殊的Bean，其作用完全是为了生产目标Bean而存在，我们通过容器获取目标Bean的时候，容器会通过工厂Bean来生产目标Bean。

#### 接口：FactoryBean

FactoryBean是一个特殊的接口，如果一个Bean实现了FactoryBean接口。那么该Bean是目标Bean的工厂Bean。我们使用容器去getBean时候，容器会去调用当前工厂Bean的getObject()方法，在getObject()中我们可以自定义初始化一个Class的实例，返回的对象作为目标Bean本身。在目标Bean初始化逻辑非常复杂的情况下，FactoryBean接口是一个非常有用的接口，它可以帮助我们去自定义一个Bean的初始化过程。这个作用和@Bean注解的方法非常相似，**区别在于FactoryBean返回的目标Bean没有SpringBean的生命周期，而@Bean返回的目标Bean是拥有完整的Bean生命周期的**。@Bean常常用于@Configuration类中，适合在Application层面的配置。但作为库的程序推荐使用FactoryBean接口的方式。以免和库的使用者的配置发生冲突。

#### 使用注意点：

对于getObject()暴露出去的实例最好在`FactoryBean`类中保存一下引用关系，因为通过`FactoryBean`的工厂Bean暴露出去的目标Bean是不存在Spring IoC容器Bean的生命周期的，如果要应用程序关闭后，目标Bean需要做一些资源回收的操作只能在工厂Bean中做，工厂Bean是拥有SpringBean的完整生命周期的，如果工厂Bean中没有缓存对目标Bean的引用关系，那么没有办法做到目标Bean的资源回收。



---

## 配置文件的读取

需要自定义读取一个properties配置文件的时候的可以使用两种方式：

上面两种方式的文件都放置在Resources目录下面

注解的方式：

```java
@Configuration
@PropertySource(value = "classpath:/my-custom-app.properties", encoding = "UTF-8")
public class MyBeanConfig {
}

```



---

## AOP编程

Spring的AOP是一个通用的切面编程封装。它封装了结合JDK动态代理和CGLIB的基于类的代理使用方式。根据不同的情况，Spring会自动的选择适合的切面实现方式和提供统一的使用方式，既通过`ProxyFactory`代理工厂类来生成代理对象。

Spring中如何生成使用AOP编程？有两种方式：一种是结合IoC容器的实现，另一种是基于编程的实现`ProxyFactory`类通过编程来生成代理类

### 基于IoC容器的实现：

1.声明Advice类，使用@Aspect

```java
/* 使用@Aspect注解来声明当前的类是一个需要被织入被代理对象方法的类定义。需要将其注入IoC容器才能正确的被Spring框架处理 */
@Aspect
@Component
public class NotVeryUsefulAspect {

}
```

2.声明Advice方法

可以使用的Advice：

- @Before：被代理方法被执行的通知
- @AfterReturning：在被代理方法执行后通知
- @AfterThrowing：在被代理方法抛出异常后通知
- @After：在被代理方法执行后，无论是否返回或者抛出异常都通知
- @Around：环绕通知，带被代理方法的前后都能被通知

```java
@Aspect
@Component
public class NotVeryUsefulAspect {
  @Around("execution(* tech.mufeng.app.controller.*.(..)")
  public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
    // start stopwatch
    Object retVal = pjp.proceed();
    // stop stopwatch
    return retVal;
  }
}
```

3.声明切点常用的几种方式：

1. execution表达式
2. @annotation表达式



### 基于编程的方式使用`ProxyFactory`方式

```java
public interface BusinessInterface {
  public String doSomething();
}

public class MyBusinessInterfaceImpl implements BusinessInterface {
  public String doSomething() {
    return "hello";
  }
}


import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
public class AroundInteceptor implements MethodInterceptor {
  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    System.err.println(invocation.getMethod().getName() + "调用之前");
    Object res = invocation.proceed();
    System.err.println(invocation.getMethod().getName() + "调用之后");
    return res;
  }
}


/* 使用JD动态代理 */
public class TestJdkDynamicProxy {
  public static void mian() {
    ProxyFactory factory = new ProxyFactory();
    factory.addInterface(BusinessInterface.class);
    factory.setTarget(new MyBusinessInterfaceImpl());
    factory.addAdvice(new AroundInteceptor());
    MyBusinessInterface tb = (MyBusinessInterface) factory.getProxy();
    tb.doSomething();
  }
}

/* 使用CBLIB生成代理对象 */
public class TestCBLIBProxy {
  public static void mian() {
    ProxyFactory factory = new ProxyFactory();
    factory.setTarget(new MyBusinessInterfaceImpl());
    factory.addAdvice(new AroundInteceptor());
    MyBusinessInterface tb = (MyBusinessInterface) factory.getProxy();
    tb.doSomething();
  }
}
```



---

## Spring使用的注意点：

### 1.  @Scope 注解中的proxyModel。
如果一个类的作用域不是singleton单例的情况下，其作为一个单例的一个依赖的情况下，必须制定代理模式为：INTERFACES或TARGET_CLASS。如果当前类是按接口类型注入的，那就可以使用INTERFACES作为代理模式，Spring会基于该接口生产动态代理类。如果当前类没有接口实现，那就需要指定代理模式为TARGET_CLASS，Spring会使用CGLIB生成代理类。因为作为单例类的依赖类，在单例类完成Bean的初始话后，属性就固定了。所以为了每次都能调用到不同Scope的Bean需要代理类来实现代理才能实现。



### 2.  自定义@ComponentScan和@Filter的使用

由于扫描一些自定义的类并把他注册到容器中

```java
@Configuration
@ComponentScan(basePackages = "org.example",
	includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
	excludeFilters = @Filter(Repository.class))
public class AppConfig {
  ...
}
```
![](https://img2020.cnblogs.com/blog/1513538/202009/1513538-20200923003658581-1659544651.png)





### 3.  @Bean注解的使用注意点，避免过早实例化

#### @Bean注解需要在@Configuration声明的类中使用

@Bean注解需要在@Configuration声明的类中使用，特别需要依赖容器中的其它Bean的时候。需特别注意依赖Bean的注入方式。如果在非@Configuration的类中使用@Bean注解方法，那么这样无法保证当前@Bean返回的目标Bean的依赖关系。需要在当前的类内部进行依赖的保证。所以建议@Bean注解需要在@Configuration声明的类中使用。



#### 使用方法参数的方式注入依赖

因为使用@Configuration的类在容器初始化之后会很早被执行，如果通过@Autowired或者@Value使用注入的话可能会导致一些意想不到的异常，建议通过方法的参数注入依赖Bean。



#### 用@Bean注解静态方法来避免过早实例化

还有要注意的是。如果使用@Bean修饰的方法返回的是自定义的BeanPostProcessor或者BeanFactoryPostPorcessor的话，需要使用静态方法来提升方法运行的优先级，因为@Configuration自己本身就是一个Bean。如果说在@Configuration实例化时候，才实例化自定义的BeanPostProcessor或者BeanFactoryPostPorcessor的话可能会导致一些Bean不能被正确的处理。

**坏的依赖注入示例**：

```java
public class CustomBeanPostProcessor implements BeanPostProcessor {
  @Override
  public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    /* some process logic */
    return bean;
  }

  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    /* some process logic */
    return bean;
  }
}


@Configuration
public class MyConfig {
  @Bean
	public CustomBean customBean() {
    return new CustomBean(customBeanDependence);
	}
	
  @Bean
  public CustomBeanPostProcessor customBeanPostProcessor() {
    return new CustomBeanPostProcessor();
  } 
}
```

**好的依赖注入示例**：

```java
@Configuration
public class MyConfig {
  
  @Bean
  public CustomBean customBean(CustomBeanDependence customDependence) {
    return new CustomBean(customDependence);
  }
  
  /* 使用static来提升执行的优先级，在实例化MyConfig之前就执行自定义BeanPostProcessor的类 */
  @Bean
  public static CustomBeanPostProcessor customBeanPostProcessor() {
    return new CustomBeanPostProcessor();
  } 
}
```

