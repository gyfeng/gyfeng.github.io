---
layout:     post
title:      "学习笔记 - Spring源码学习三"
subtitle:   "Bean初始化过程，BeanFactory的getBean方法"
header-img: "img/in-post/spring-lesson/spring-logo.jpg"
header-mask: 0.2
catalog:    true
tags:
    - Spring
---
> 上篇笔记学习了[Spring从Xml加载Bean的过程](http://blog.codedoge.com/2017/05/07/lesson-spring-source-ioc/)


## Bean的创建过程
&#8195;&#8195;通过上面的分析，已经知道XmlBeanFactory是通过BeanDefinitionDocumentReader在初始化的时候需要依赖BeanDefinitionRegistry解析xml文件并将BeanDefinition注册到XmlBeanFactory中。那么，BeanDefinition又是怎么转变为我们使用的Bean呢？国为XmlBeanFactory中没有定义单独的初始化方法，所以猜测应该是在调用`getBean`方法的时候再去初始化的。而且，所有`getBean`方法，最终都会调用到`AbstractBeanFactory.doGetBean`方法。那就从`doGetBean`方法开始分析吧。来人啊，上源码：
```
protected <T> T doGetBean(final String name, final Class<T> requiredType,
  final Object[] args, boolean typeCheckOnly){

  final String beanName = transformedBeanName(name);
  Object bean;
  
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }

  else {
    if (isPrototypeCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(beanName);
    }
    BeanFactory parentBeanFactory = getParentBeanFactory();
    if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
      String nameToLookup = originalBeanName(name);
      if (args != null) {
        return (T) parentBeanFactory.getBean(nameToLookup, args);
      }
      else {
        return parentBeanFactory.getBean(nameToLookup, requiredType);
      }
    }
    if (!typeCheckOnly) {
      markBeanAsCreated(beanName);
    }
    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    checkMergedBeanDefinition(mbd, beanName, args);
    String[] dependsOn = mbd.getDependsOn();
    if (dependsOn != null) {
      for (String dependsOnBean : dependsOn) {
        getBean(dependsOnBean);
        registerDependentBean(dependsOnBean, beanName);
      }
    }

    if (mbd.isSingleton()) {
      sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
        public Object getObject() throws BeansException {
            return createBean(beanName, mbd, args);
        }
      });
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }

    else if (mbd.isPrototype()) {
      Object prototypeInstance = null;
      try {
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
      }
      finally {
        afterPrototypeCreation(beanName);
      }
      bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
    }
    else {
      String scopeName = mbd.getScope();
      final Scope scope = this.scopes.get(scopeName);
      if (scope == null) {
        throw new IllegalStateException("No Scope registered for scope '"
      + scopeName + "'");
      }
      Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
        public Object getObject() throws BeansException {
          beforePrototypeCreation(beanName);
          try {
            return createBean(beanName, mbd, args);
          }
          finally {
            afterPrototypeCreation(beanName);
          }
        }
      });
      bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
    }
  }
  if (requiredType != null && bean != null 
    && !requiredType.isAssignableFrom(bean.getClass())) {
    return getTypeConverter().convertIfNecessary(bean, requiredType);
  }
  return (T) bean;
}
```
&#8195;&#8195;`doGetBean`方法有多个参数，大概作用如下：
> 1. name 要获取的Bean的名称（必填）
> 2. requiredType 要获取Bean的类型
> 3. args 构造bean时的参数，注意，只有多例时需要
> 4. typeCheckOnly 还没看明白，类型检查？？

&#8195;&#8195;从`doGetBean`方法处理流程来看，大概分为这几步。
> 1. 获取已定义的单例对象（按名字获取），若获取到，调用getObjectForBeanInstance处理对象，然后返回。否则继续
> 2. 若父容器存在且当前容器中不存在Bean的定义时，从父容器中获取。否则继续
> 3. 加载依赖的Bean对象
> 4. 根据不同的作用域调用`createBean`方法创建对象。
> 5. 调用getObjectForBeanInstance处理对象，这一步主要是处理FactoryBean等情况

&#8195;&#8195;*从上面的流程中，可以看出均调用了`createBean`和`getObjectForBeanInstance`方法，区别只是不同的作用域下对结果的处理不一样，比如singleton时会做一些缓存工作*。
&#8195;&#8195;在`createBean`中(*此方法在AbstractAutowireCapableBeanFactory中实现*)。
```
protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args){
  resolveBeanClass(mbd, beanName);
  mbd.prepareMethodOverrides();
  // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
  Object bean = resolveBeforeInstantiation(beanName, mbd);
  if (bean != null) {
    return bean;
  }
  Object beanInstance = doCreateBean(beanName, mbd, args);
  return beanInstance;
}
```
&#8195;&#8195;哇，创建Bean居然有两个方法。从注释上看`resolveBeforeInstantiation`方法主要用于代理生成对象（注：`从实际调试跟踪代码过程中没有发现从这个方法生成对象`）。在`doCreateBean` 方法中，
```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
  // Instantiate the bean.
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
  }
  if (instanceWrapper == null) {
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }
  final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
  Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

  // Allow post-processors to modify the merged bean definition.
  synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
      applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
      mbd.postProcessed = true;
    }
  }

  // Initialize the bean instance.
  Object exposedObject = bean;
  populateBean(beanName, mbd, instanceWrapper);
  if (exposedObject != null) {
    exposedObject = initializeBean(beanName, exposedObject, mbd);
  }
  registerDisposableBeanIfNecessary(beanName, bean, mbd);
  return exposedObject;
}
```
