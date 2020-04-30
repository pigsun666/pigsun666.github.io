---
layout:     post
title:      Spring源码阅读
subtitle:   (三)BeanDefinition
date:       2020-4-30
author:     pigsun
header-img: img/post-bg-spring.jpeg
catalog: true
tags:
    - Spring
    - Spring源码
---
# BeanDefinition



## 什么是BeanDefinition

贴出官网的描述

> A Spring IoC container manages one or more beans. These beans are created with the configuration metadata that you supply to the container (for example, in the form of XML `` definitions).
>
> Within the container itself, these bean definitions are represented as `BeanDefinition` objects, which contain (among other information) the following metadata:
>
> - A package-qualified class name: typically, the actual implementation class of the bean being defined.
> - Bean behavioral configuration elements, which state how the bean should behave in the container (scope, lifecycle callbacks, and so forth).
> - References to other beans that are needed for the bean to do its work. These references are also called collaborators or dependencies.
> - Other configuration settings to set in the newly created object — for example, the size limit of the pool or the number of connections to use in a bean that manages a connection pool.

我的理解是

SpringIOC容器统一管理一个或者多个Bean,这些Bean是我们提供给容器的配置元数据创建的(xml配置或者注解方式),对于容器来说，这些bean都会被解释成一个BeanDefinition对象，BeanDefinition是对bean的描述，其中包含

* 包限定的类名，通常是Bean的实现类
* Bean行为配置元素，比如Bean的作用于，生命周期回调等等
* Bean之间的依赖关系
* 其他配置 比如property

前文有提过普通对象和spring bean的区别，下面用图表示出来

![](https://pic-go-pigsun.oss-cn-shanghai.aliyuncs.com/pigGo/20200418115531.png)

`BeanDefinition`是一个接口，下面列出方法分析

```java
// 获取父BeanDefinition名字
@Nullable
String getParentName();

//设置Bean的className
void setBeanClassName(@Nullable String beanClassName);

//获取Bean的className
@Nullable
String getBeanClassName();

//// Bean的作用域，不考虑web容器，主要两种，单例/原型，见官网中1.5内容
void setScope(@Nullable String scope);

/**
 * Return the name of the current target scope for this bean,
 * or {@code null} if not known yet.
 */
@Nullable
String getScope();

// 是否进行懒加载
void setLazyInit(boolean lazyInit);

boolean isLazyInit();

// 是否需要等待指定的bean创建完之后再创建
void setDependsOn(@Nullable String... dependsOn);

@Nullable
String[] getDependsOn();

// 是否作为自动注入的候选对象
void setAutowireCandidate(boolean autowireCandidate);

boolean isAutowireCandidate();

// 是否作为主选的bean
void setPrimary(boolean primary);

/**
 * Return whether this bean is a primary autowire candidate.
 */
boolean isPrimary();

// 创建这个bean的类的名称 工厂bean名称
void setFactoryBeanName(@Nullable String factoryBeanName);

/**
 * Return the factory bean name, if any.
 */
@Nullable
String getFactoryBeanName();

// 创建这个bean的方法的名称 工厂方法名臣
void setFactoryMethodName(@Nullable String factoryMethodName);

String getFactoryMethodName();

// 构造函数的参数
ConstructorArgumentValues getConstructorArgumentValues();


// setter方法的参数   xml配置里面<property value>标签
MutablePropertyValues getPropertyValues();


// 生命周期回调方法，在bean完成属性注入后调用
void setInitMethodName(@Nullable String initMethodName);


@Nullable
String getInitMethodName();

// 生命周期回调方法，在bean被销毁时调用
void setDestroyMethodName(@Nullable String destroyMethodName);


@Nullable
String getDestroyMethodName();

// Spring可以对bd设置不同的角色,了解即可，不重要
// 用户定义 int ROLE_APPLICATION = 0;
// 某些复杂的配置    int ROLE_SUPPORT = 1;
// 完全内部使用   int ROLE_INFRASTRUCTURE = 2;
int getRole();

void setDescription(@Nullable String description);

// bean的描述，没有什么实际含义
@Nullable
String getDescription();

// 根据scope判断是否是单例
boolean isSingleton();

// 根据scope判断是否是原型
boolean isPrototype();

//是不是抽象类
boolean isAbstract();

@Nullable
String getResourceDescription();

// cglib代理前的BeanDefinition
@Nullable
BeanDefinition getOriginatingBeanDefinition();
```

## `BeanDefinition` 继承关系

![](https://pic-go-pigsun.oss-cn-shanghai.aliyuncs.com/pigGo/GenericBeanDefinition.png)

### BeanDefinition

* `org.springframework.core.AttributeAccessor`

> ```
> Interface defining a generic contract for attaching and accessing metadata to/from arbitrary objects.
> ```

接口，定义将元数据附加到任意对象或从任意对象访问元数据的通用协定

```java
	  void setAttribute(String name, @Nullable Object value);

      Object getAttribute(String name);

      Object removeAttribute(String name);

      boolean hasAttribute(String name);

      String[] attributeNames();
```

看方法结构 是提供获取属性和设置属性的方法



在`BeanDefinition`的体系中 直接实现了`AttributeAccessor `,是org.springframework.core.AttributeAccessorSupport(抽象类)

```java
public abstract class AttributeAccessorSupport implements AttributeAccessor, Serializable {

   /** Map with String keys and Object values. */
   private final Map<String, Object> attributes = new LinkedHashMap<>();


   @Override
   public void setAttribute(String name, @Nullable Object value) {
      Assert.notNull(name, "Name must not be null");
      if (value != null) {
         this.attributes.put(name, value);
      }
      else {
         removeAttribute(name);
      }
   }
```

部分代码如上，可以看到 当前类内部维护了一个map，设置为私有的，然后提供对外的接口操作map。



* `org.springframework.beans.BeanMetadataElement`

> Interface to be implemented by bean metadata elements that carry a configuration source object.

将由承载配置源对象的bean元数据元素实现的接口。

这个类是个顶级接口，只有一个方法

```java
@Nullable
default Object getSource() {
   return null;
}
```

我们可以理解为，当我们通过注解的方式定义了一个IndexService时，那么此时的IndexService对应的`BeanDefinition`通过getSource方法返回的就是IndexService.class这个文件对应的一个File对象。

如果我们通过@Bean方式定义了一个IndexService的话，那么此时的source是被@Bean注解所标注的一个Mehthod对象。



### AbstractBeanDefinition(抽象类)

当前类是`BeanDefinition`的部分实现，同时定义了一系列常量和默认字段。由于BeanDefinition接口过于顶层，如果我们直接依赖`BeanDefinition`这个接口直接去创建其实现类的话过于麻烦，所以通过`AbstractBeanDefinition`做个下沉，并且给一些属性设置默认值，例如:

```java
public static final int AUTOWIRE_NO = AutowireCapableBeanFactory.AUTOWIRE_NO;

public static final int AUTOWIRE_BY_NAME = AutowireCapableBeanFactory.AUTOWIRE_BY_NAME;

public static final int AUTOWIRE_BY_TYPE = AutowireCapableBeanFactory.AUTOWIRE_BY_TYPE;

public static final int AUTOWIRE_CONSTRUCTOR = AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR;
```

这样设计更加方便我们去创建子类，下面要介绍它的子类

* `org.springframework.beans.factory.support.GenericBeanDefinition`

最常规的BeanDefinition，替代了原来的ChildBeanDefinition。ChildBeanDefinition在实例化的时候必须指定一个ParentName，就是父BeanDefinition的name，GenericBeanDefinition不需要。我们代码中配置的Bean基本都是这个类型的。（除了@Bean）

* `org.springframework.beans.factory.support.RootBeanDefinition`
  * spring启动的时候会实例化几个初始化的`BeanDefinition`，这几个`BeanDefinition`都是`RootBeanDefinition`
  * Spring在合并`BeanDefinition`的时候返回的都是用RootBeanDefinition接收的
  * 我们通过@Bean注解配置的bean，解析出来的`BeanDefinition`都是`RootBeanDefinition`(实际上是其子类`ConfigurationClassBeanDefinition`)

* `org.springframework.beans.factory.support.ChildBeanDefinition`

已经被`GenericBeanDefinition`替代



### `AnnotatedBeanDefinition`(接口)

该类直接继承的`BeanDefinition`  新增了两个方法，是对`BeanDefinition`的增强。

```java
AnnotationMetadata getMetadata();

@Nullable
MethodMetadata getFactoryMethodMetadata();
```

* `getMetadata()` 主要用于获取注解元素据。从接口的命名上我们也能看出，这类主要用于保存通过注解方式定义的bean所对应的`BeanDefinition`。所以它多提供了一个关于获取注解信息的方法
* `getFactoryMethodMetadata()`这个方法跟我们的`@Bean`注解相关。当我们在一个配置类中使用了`@Bean`注解时，被`@Bean`注解标记的方法，就被解析成了`FactoryMethodMetadata`。



根据前文的类图，实现当前接口的有三个类(`ConfigurationClassBeanDefinition` /`ScannedGenericBeanDefinition` /`AnnotatedGenericBeanDefinition`)



#### 1:`ConfigurationClassBeanDefinition`

通过`@Bean`的方式配置的Bean为`ConfigurationClassBeanDefinition`

#### 2:`ScannedGenericBeanDefinition` 

**该类不仅实现了**`AnnotatedBeanDefinition` 还继承了`GenericBeanDefinition`

都过注解扫描的类，如`@Service`,`@Compent`等方式配置的Bean都是`ScannedGenericBeanDefinition`

#### 3:`AnnotatedGenericBeanDefinition`

**该类不仅实现了**`AnnotatedBeanDefinition` 还继承了`GenericBeanDefinition`

在我们启动Spring容器的时候注册的配置类，配置类就会被解析成`AnnotatedGenericBeanDefinition`

对象。

ac.register(AppConfig.class)最终就会调用到这个方法，这里直接new出来`AnnotatedGenericBeanDefinition`

```java
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
      @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
      @Nullable BeanDefinitionCustomizer[] customizers) {

   AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
   if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
      return;
   }
```



## 总结



什么是BeanDefinition？BeanDefinition就是Spring创建Bean的建模对象，是对Bean的抽象。

![](https://pic-go-pigsun.oss-cn-shanghai.aliyuncs.com/pigGo/20200418154003.png)