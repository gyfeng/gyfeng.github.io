---
layout:     post
title:      "学习笔记 - Spring源码学习一"
subtitle:   "Ioc结构体系"
header-img: "img/in-post/spring-lesson/spring-logo.jpg"
header-mask: 0.2
catalog:    true
tags:
    - Spring
---
> 学习Spring源码，从最基本的Ico容器开始，逐步了解Spring的整个概况。学习，是为了更大的进步。

## 从简单开始
&#8195;&#8195;Ioc是Spring最核心的功能，也是Spring一切功能的起点。今天，将从最基本的Ioc开始学习，逐步学习了解神秘的Spring。  
&#8195;&#8195;XmlBeanFactory是Spring提供的最简单的Ioc容器之一，我最喜欢从简单开始了，那么现在将从XmlBeanFactory的结构体系中逐步进行分析。

## XmlBeanFactory的类继承体系
![Spring XmlBeanFactory类图](/img/in-post/spring-lesson/ioc/class-XmlBeanFactory.png)
&#8195;&#8195;在XmlBeanFactory的类图中，我们可以看到BeanFactory和AliasRegistry占据最高点。  
&#8195;&#8195;BeanFactory定义了一系列获取Bean的方法，AliasRegistry(BeanDefinitionRegistry)提供的注册Bean的功能，而没有在类图中提到的BeanDefinitionReader则定义了从配置文件中加载Bean定义到BeanDefinitionRegistry中。三者配合，组成了Spring的核心框架。当然，具体功能还需要具体类来实现！
![Spring XmlBeanFactory类图](/img/in-post/spring-lesson/ioc/class-XmlBeanFactory-2.png)
#### BeanFactory体系
&#8195;&#8195;BeanFactory体系的类定义了一系列获取Bean的方法。BeanFactory位置该体系的结构树的顶端，最主要的方法就是`getBean`，方法从容器中返回特定的Bean,BeanFactory的功能通过其他接口得到不断扩展。下面对该体系涉及的其他接口进行说明：
- ListableBeanFactory：该接口定义了访问容器中Bean的若干方法，比如获取Bean个数，获取容器中所有Bean的名称等；
- HierarchicalBeanFactory：父子级联容器接口，子容器可能通过接口访问父容器，SpringMVC 就是通过父子容器实现Bean的隔离；
- ConfigurableBeanFactory：定义了ClassLoader、属性编辑器、容器初始化后置处理器等方法。是一个重要的接口，增加了容器的可定制性；
- AutowireCapableBeanFactory、：定义了将容器中的Bean按某种规则（如按名字、类型匹配）进行自动装配的方法；

#### Registry体系
&#8195;&#8195;Registry体系的接口提供了注册Bean的方法，BeanFactory体系是从容器中获取Bean,Registry体系则是向容器中注册Bean。
![Spring Registry体系类图](/img/in-post/spring-lesson/ioc/class-Registry.png)
&#8195;&#8195;该体系中主要是一系列的`register`的方法，向容器注册Bean的相关信息。同样，也对该体系下涉及的其他接口进行说明：
- AliasRegistry：提供向容器注册Bean别名的功能；
- SingletonBeanRegistry：提供向容器注册单例Bean的方法；
- BeanDefinitionRegistry：提供了向容器注册BeanDefinition的方法，Spring配置中每一个&lt;bean&gt;节点元素都通过一个BeanDefinition对象表示，用于描述Bean的配置信息。

#### BeanDefinitionReader体系
&#8195;&#8195;BeanDefinitionReader主要定义了从资源文件是加载bean定义的方法，主要是一系列`loadBeanDefinition`方法。
![Spring BeanDefinitionReader体系类图](/img/in-post/spring-lesson/ioc/class-BeanDefinitionReader.png)

## 总结
&#8195;&#8195;BeanFactory、Registry以及BeanDefinitionReader组成了Spring Ioc容器的核心，BeanFactory定义了从Ioc容器中获取Bean的方法，Registry定义向容器注册Bean的方法，而BeanDefinitionReader定义了从资源文件读取bean定义的方法。相互组合形成了Spring最核心的Ioc功能。这样的模块化设计也为扩展留出了巨大空间，如果需要扩展获取Bean的方法，需要扩展BeanFactory，如果需要换一种Bean的注册或存储方法，需要扩展Registry，如果需要扩展从其他类型的资源文件加载bean定义，只需要扩展BeanDefinitionReader即可。Spring的这种模块化设计理念提供了充足的扩展空间，Spring后面的很多扩展功能都是建立在这种扩展方式之上。  

## 参考文档
[1] 陈开雄.Spring 3.X企业应用开发实战.电子工业出版社，2012.2:
