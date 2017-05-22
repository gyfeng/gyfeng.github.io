---
layout:     post
title:      "学习笔记 - Spring源码学习四"
subtitle:   "ApplicationContext及其初始化过程"
header-img: "img/in-post/spring-lesson/spring-logo.jpg"
header-mask: 0.2
catalog:    true
tags:
    - Spring
---

> &#8195;&#8195;前两篇笔记学习了[Spring从Xml加载Bean Definition的过程](http://blog.codedoge.com/2017/05/08/lesson-spring-source-beandefinitionreader/)以及[Bean初始化过程](http://blog.codedoge.com/2017/05/10/lesson-spring-source-beaninit/)，了解到了在AbstractBeanFactory及其子类中定义了Spring Ioc的骨架，已经可以使用使用Ico的功能，但是一些特定的组件还需要手工进行注册，而在ApplicationContext中，已经将这些组件自动为我们装配好了。在《Spring 3.X企业应用开发实战》一书中，更是形象地把BeanFactory比喻为发动机，把ApplicationContext比喻为汽车，BeanFactory是核心，而ApplicationContext是在BeanFactory上组合各种组件，更强大，更易用，接下来，学习ApplicationContext的初始化过程，学习ApplicationContext是怎么在BeanFactory上进行增强的。

## ApplicationContext继承体系
先上一张大图，ApplicationContext的继承结构体系。
![ApplicationContext继承结构体系](/img/in-post/spring-lesson/applicationcontextinit/class-ApplicationContext.png)
&#8195;&#8195;从上图可以看出，ApplicationContext继承了ListableBeanFactory和HierarchicalBeanFactory接口，在此基础上，还通过多个其他接口扩展了BeanFactory的功能。
- ApplicationEventPublisher：让容器拥有发布应用上下文事件的功能，包括容器启动、关闭等事件，并对事件进行响应处理。在ApplicationContext抽象实现类AbstractApplicationContext中，可以发现存在一个ApplicationEventMulicaster，它负责保存所有监听器，以便在容器产生上下文事件时通知这些事件监听者；
- MessageSource：为容器提供国际化消息访问的功能；
- 其他 ... ...

再上一张大图，ApplicationContext的全家福！
![ApplicationContext全家福](/img/in-post/spring-lesson/applicationcontextinit/class-ApplicationContext-all.png)
&#8195;&#8195;从全家福中可以看出，抽象类AbstractApplicationContext在ApplicationContext家族中拥有非常重要的地位，那我们的分析就可以从其慢慢展开。

## AbstractApplicationContext结构
不能免俗，还是先上一张大图！
![AbstractApplicationContext结构](/img/in-post/spring-lesson/applicationcontextinit/class-AbstractApplicationContext.png)
&#8195;&#8195;从图中可以看出，AbstractApplicationContext已经对ApplicationContext进行了扩展和实现，大概如下：
- 持有ApplicationContext类型的引用作为父容器，实现ApplicationContext的父子容器；
- 持有ResourcePatternResolver及MessageSource的引用，主要用于加载资源和国际化功能。

#### 重要方法
&#8195;&#8195;AbstractApplicationContext中实现了许多重要的接口方法，也定义了许多抽象方法用于子类扩展：
- getBean系列方法：实现了BeanFactory中定义的从容器中获取Bean的方法，注意在实现中是将该方法委托给getBeanFactory()方法获取到的BeanFactory，可见实现者只是想把ApplicationContext做为BeanFactory的视图，而非直接用于存储Bean对象，也使用refresh方法实现更加简单；
- refresh：实现了ConfigurableApplicationContext中的refresh方法，用于刷新容器；
- getBeanFactory：ConfigurableApplicationContext定义的抽象方法，这里并未进行实现。此方法用于返回此ApplicationContext的内部BeanFactory，需要子类具体实现；
- getMessage系列方法：实现了MessageSource中获取国际化信息的方法，在此类中也是委托给getMessageSource()获取到的MessageSoure；
- 其他 ... ... *都挺重要的，慢慢分析吧！*

## 初始化
&#8195;&#8195;在ApplicationContext实现类的初始化过程中，一般都会先设定Bean定义文件的位置，再调用refresh方法进行初始化，现在我们就对refresh方法进行深入分析。先上源码
```
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    // 准备上下文进行刷新工作，设置其启动日期、活动标志以及执行属性源的初始化。
    prepareRefresh();
    // 通知子类刷新工厂并返回
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
    // 配置工厂的标准上下文特征，如上下文的ClassLoader
    prepareBeanFactory(beanFactory);
    try {
      // 允许在上下文子类中对bean工厂进行后处理
      postProcessBeanFactory(beanFactory);
      // 调用工厂后处理器
      invokeBeanFactoryPostProcessors(beanFactory);侵权
      // 注册Bean处理器
      registerBeanPostProcessors(beanFactory);
      // 实例化消息源
      initMessageSource();
      // 初始化事件广播器
      initApplicationEventMulticaster();
      // 由子类实现用于初始化其他的特殊Bean
      onRefresh();
      // 注册事件监听器
      registerListeners();
      // 初始化所有单例Bean
      finishBeanFactoryInitialization(beanFactory);
      // 完成refresh并发布容器refresh事件
      finishRefresh();
    }catch (BeansException ex) {
      // 刷新失败，销毁已创建的Bean
      destroyBeans();
      // 重置标记
      cancelRefresh(ex);
      throw ex;
    }
  }
}
```
&#8195;&#8195;整个初始化过程(refresh)都是围绕BeanFactory进行的，都是在各位不同的阶段进行一些特殊组件的注册工作。我们再一步一下地展开分析。

#### prepareRefresh
&#8195;&#8195;准备上下文进行刷新工作，设置其启动日期、活动标志以及执行属性源的初始化。

#### obtainFreshBeanFactory
&#8195;&#8195;通知子类刷新工厂并返回工厂，在方法实现中，先调用`refreshBeanFactory`抽象方法(子类实现)刷新工厂，再调用`getBeanFactory`方法返回工厂，方法`refreshBeanFactory`实现源码如下（*AbstractRefreshableApplicationContext实现*）：
```
if (hasBeanFactory()) {
  destroyBeans();
  closeBeanFactory();
}
DefaultListableBeanFactory beanFactory = createBeanFactory();
beanFactory.setSerializationId(getId());
customizeBeanFactory(beanFactory);
loadBeanDefinitions(beanFactory);
synchronized (this.beanFactoryMonitor) {
  this.beanFactory = beanFactory;
}
```
从源码上看，`refreshBeanFactory`方法的主要流程如下：
1. 若已经存在beanFactory，则销毁其中注册的Bean对象；
2. 创建新的beanFactory(`new DefaultListableBeanFactory(getInternalParentBeanFactory())`)
3. 设置工厂ID；
4. 定制BeanFactory的属性；
5. 加载BeanDefinition到当前BeanFactory中（还记得*BeanDefinitionReader*吧）；

#### prepareBeanFactory
&#8195;&#8195;配置工厂的标准上下文特征，如上下文的ClassLoader。

#### postProcessBeanFactory
&#8195;&#8195;扩展方法，允许在上下文子类中对bean工厂进行后处理。

#### invokeBeanFactoryPostProcessors
&#8195;&#8195;调用如下工厂后处理器的`postProcessBeanFactory`方法：
- 手工调用`addBeanFactoryPostProcessor`方法注册的FactoryPostProcessor；
- 容器中注册的`BeanDefinitionRegistryPostProcessor`类型的处理器；
- 容器中注册的`BeanFactoryPostProcessor`类型的处理器；

#### registerBeanPostProcessors
&#8195;&#8195;注册Bean处理器，从容器是获取`BeanPostProcessor`类型的Bean，并调用`beanFactory.addBeanPostProcessor`方法注册到BeanFactory中。

#### initMessageSource
&#8195;&#8195;实例化消息源，在方法中，首先获取工厂中注册的名为messageSource的Bean，若存在，则将该Bean注册为当前消息源，否则使用`new DelegatingMessageSource()`为当前消息源，并注册到容器中。

#### initApplicationEventMulticaster
&#8195;&#8195;初始化事件广播器。

#### onRefresh
&#8195;&#8195;扩展方法，由子类实现用于初始化其他的特殊Bean。

#### registerListeners
&#8195;&#8195;注册事件监听器。将容器手工设置的ApplicationListener和工厂中ApplicationListener类型的Bean注册到事件广播器中。

#### finishBeanFactoryInitialization
&#8195;&#8195;调用`beanFactory.preInstantiateSingletons`方法初始化单例Bean，具体参见上节学习内容[Bean初始化过程](http://blog.codedoge.com/2017/05/10/lesson-spring-source-beaninit/)。

#### finishRefresh
&#8195;&#8195;发布refresh事件。

## 总结
&#8195;&#8195;今天学习的ApplicationContext的体系结构及初始化过程，ApplicationContext的继承了ListableBeanFactory、HierarchicalBeanFactory接口拥有了BeanFactory的功能，同时也继承ApplicationEventPublisher、MessageSource、ResourcePatternResolver等，使得ApplicationContext的功能更加全面。在ApplicationContext实现类中，拥有一个BeanFactory类型的实例，在容器中的大部分操作都是围绕该BeanFactory实例进行的，刷新功能也是通过创建另一个BeanFactory实现。  
&#8195;&#8195;初始化过程中，ApplicationContext向BeanFactory注册了大量组件，增强了BeanFactory的功能，比如BeanFactoryPostProcessor、BeanPostProcessor等，这些特殊的组件可通过ApplicationContext相应的add或set方法提供，也获取了BeanFactory中注册的相应类型组件，所以要扩展相应功能，可手工调用add可set方法，也可通过向容器注册Bean实例的方式。  
&#8195;&#8195;ApplicationContext是Spring中的高级容器，提供了比BeanFactory更强大功能和更加易于使用，一般在使用Ioc容器的时候都是使用ApplicationContext而非直接BeanFactory。
