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
> 上篇笔记学习了[Spring从Xml加载Bean Definition的过程](http://blog.codedoge.com/2017/05/07/lesson-spring-source-ioc/)，了解到了Spring是如何将Xml文件中定义的Bean加载到容器中的，今天，将学习Spring是如何将BeanDefinition生成我们所需要Bean的过程。

## Bean创建的时机
&#8195;&#8195;Spring不会默然自动将BeanDefinition转换我们所需要Bean，而是在调用`getBean`系列方法时再去做Bean初始化的操作，虽然这看起来有点不符合我们的认知，因为在ApplicationContext中可以把单例的Bean提前初始化，在我们手工`getBean`时就已经初始化好了，其实这是在ApplicationContext初始化时，不断调用`getBean`方法将单例的Bean进行了初始化工作，使得我们在使用的时候不用经过繁琐创建过程。(PS:*`DefaultListableBeanFactory。preInstantiateSingletons`方法可以可以预加载单例Bean，`ApplicationContext`中就是调用该方法实现预加载单例Bean的*)  
&#8195;&#8195;在BeanFactory中，最终所有的`getBean`方法都会调用`AbstractBeanFactory.doGetBean`方法获取，那就从该方法入手进行分析。
```
public Object getBean(String name) {
  return doGetBean(name, null, null, false);
}
public <T> T getBean(String name, Class<T> requiredType) {
  return doGetBean(name, requiredType, null, false);
}
public Object getBean(String name, Object... args){
  return doGetBean(name, null, args, false);
}
public <T> T getBean(String name, Class<T> requiredType, Object... args) {
  return doGetBean(name, requiredType, args, false);
}
```
## doGetBean方法分析
&#8195;&#8195;通过上面的分析，已经知道最终Spring会调用`AbstractBeanFactory.doGetBean`方法来获取bean，那在这一节，就从这个方法开始分析。来人啊，上源码：
```
protected <T> T doGetBean(final String name, final Class<T> requiredType,
                          final Object[] args, boolean typeCheckOnly){
}
```
`doGetBean`方法有多个参数，作用如下：
> 1. name 要获取的Bean的名称（必填）,可以是别名或FactoryBean的名称；
> 2. requiredType 获取的实例要转换成的类型；
> 3. args 构造bean时的参数，注意，只有多例时需要

从`doGetBean`方法处理流程来看，大概分以下几步：
#### 1. BeanName处理，处理别名及FactoryBean Name
在调用`doGetBean`方法时，首先会去处理BeanName，以找到真正的原始Bean Id。
```
final String beanName = transformedBeanName(name);
```
```
protected String transformedBeanName(String name) {
　　return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}
```
`BeanFactoryUtils.transformedBeanName`方法处理FactoryBean Name
```
public static String transformedBeanName(String name) {
  String beanName = name;
  while (beanName.startsWith("&")) {
    beanName = beanName.substring("&");
  }
  return beanName;
}
```
`SimpleAliasRegistry.canonicalName`方法处理别名
```
public String canonicalName(String name) {
  String canonicalName = name;
  String resolvedName;
  do {
    resolvedName = this.aliasMap.get(canonicalName);
    if (resolvedName != null) {
      canonicalName = resolvedName;
    }
  }
  while (resolvedName != null);
  return canonicalName;
}
```

#### 2. 已创建单例Bean对象直接从容器获取
判断这个beanName的bean是否已经创建，那么直接使用，不需要再次进行创建。
```
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
  bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}
```
&#8195;&#8195;这里调用`DefaultSingletonBeanRegistry.getSingleton`方法从容器中获取已实例化的Bean，已实例化的单例Bean是放到Map类型的singletonObjects对象中（*这里使用了一个巧妙的设计，来解决单例循环引用的问题，后续再行分析*）。`getObjectForBeanInstance`中，若从容器中获取到的实例类型是FactoryBean，则调用`FactoryBean.getObject`获取真正的Bean对象。

#### 3. 父容器加载
若当前Factory容器中不存在当前Bean的定义，则交由父容器处理。
```
// Check if bean definition exists in this factory.
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
  // Not found -> check parent.
  String nameToLookup = originalBeanName(name);
  if (args != null) {
    // Delegation to parent with explicit args.
    return (T) parentBeanFactory.getBean(nameToLookup, args);
  } else {
    // No args -> delegate to standard getBean method.
    return parentBeanFactory.getBean(nameToLookup, requiredType);
  }
}
```

#### 4. 获取BeanDefinition
&#8195;&#8195;接下来获取BeanDefiniton，并且验证该BeanDefinition是否可实例化，在`getMergedLocalBeanDefinition`方法中处理了parent Bean，`checkMergedBeanDefinition`验证Bean是否可实例化，*Bean定义必须是非抽象的，并且非多例Bean不能有实例化参数*
```
final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
checkMergedBeanDefinition(mbd, beanName, args);
```
```
protected void checkMergedBeanDefinition(RootBeanDefinition mbd, String beanName, Object[] args)　{
  // check if bean definition is not abstract
  if (mbd.isAbstract()) {
    throw new BeanIsAbstractException(beanName);
  }
  // Check validity of the usage of the args parameter. This can
  // only be used for prototypes constructed via a factory method.
  if (args != null && !mbd.isPrototype()) {
    throw new BeanDefinitionStoreException("Can only specify arguments for the getBean method when referring to a prototype bean definition");
  }
}
```

##### 5. 加载依赖Bean
&#8195;&#8195;加载依赖的Bean，递归调用`getBean`方法加载依赖的Bean（*注意，这里没有处理循环依赖的情况*）。
```
// Guarantee initialization of beans that the current bean depends on.
String[] dependsOn = mbd.getDependsOn();
if (dependsOn != null) {
  for (String dependsOnBean : dependsOn) {
    getBean(dependsOnBean);
    registerDependentBean(dependsOnBean, beanName);
  }
}
```

#### 6. 实例化Bean
&#8195;&#8195;在这一步，会根据不同的作用域类型创建不同的Bean，最终都会调用`createBean`方法实例化Bean。  
**1). 单例对象的创建**  
&#8195;&#8195;单例对象的创建是调用`DefaultSingletonBeanRegistry.getSingleton(String,ObjectFactory)`方法实现，在方法中Bean创建完成后会把Bean添加到容器中。
```
if (mbd.isSingleton()) {
  sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
    public Object getObject() throws BeansException {
      try {
        return createBean(beanName, mbd, args);
      }
      catch (BeansException ex) {
        destroySingleton(beanName);
        throw ex;
      }
    }
  });
  bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

**2). Prototype对象的创建**  
&#8195;&#8195;多例类型的Bean，直接调用`createBean`方法进行创建。
```
if (mbd.isPrototype()) {
  // It's a prototype -> create a new instance.
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
```
&#8195;&#8195;在创建之前的`beforePrototypeCreation`中，使用ThreadLocal记录了正在创建的Prototype Bean，还记得创建开始的一段代码吗？如果要创建的对象已经在ThreadLocal中记录，表明有循环引用情况，这在prototype类型中是不应该的，否则会无限制地创建下去。
```
if (isPrototypeCurrentlyInCreation(beanName)) {
  throw new BeanCurrentlyInCreationException(beanName);
}
```
**3). 其他作用域类型对象的创建**  
&#8195;&#8195;其他类型的Bean，会根据作用域名称获取对应的作用域Scope对象，通过`scope.get`进行对象的创建，在Spring中，request、session、 global session等作用域类型对象的创建都是通过Scope来实现的。  
&#8195;&#8195;和Prototype类型一样，其他作用域类型的Bean也不允许循环引用。
```
String scopeName = mbd.getScope();
final Scope scope = this.scopes.get(scopeName);
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
```
#### 7. 类型转换
若获取到的Bean类型和所需类型不一致，则调用手工注册的编辑器进行加工处理。
```
if (requiredType != null && bean != null 
    && !requiredType.isAssignableFrom(bean.getClass())) {
  return getTypeConverter().convertIfNecessary(bean, requiredType);
}
```

## createBean方法创建对象分析
&#8195;&#8195;`createBean`方法最终是在`AbstractAutowireCapableBeanFactory`类中实现。实现代码如下：
```
protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args){
  resolveBeanClass(mbd, beanName);

  // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
  Object bean = resolveBeforeInstantiation(beanName, mbd);
  if (bean != null) {
    return bean;
  }
  Object beanInstance = doCreateBean(beanName, mbd, args);
  return beanInstance;
}
```
在`createBean`方法中，创建Bean有如下步骤：　　

> 1. 确保class可以被正确解析，通过`resolveBeanClass`方法实现，因为在Bean定义的时候可能没有具体的class，而只提供className，或者一些被加密过class，都是在这个方法中进行处理。
> 2. 给`InstantiationAwareBeanPostProcessor`实例化Bean一个机会，如果通过`InstantiationAwareBeanPostProcessor`实例化对象成功，则不经过Spring后续的处理，比如自动注入等功能。在一般情况下，此方法会返回null。
> 3. 使用`doCreateBean`方法实质性的创建bean，在`doCreateBean`方法中，创建的Bean会经过完整的流程。

## doCreateBean方法分析
#### 1. 实例化Bean
在doCreateBean方法中，首先会调用`BeanWrapper`的方法实例化Bean。`BeanWrapper`是Spring提供的分析和操作JavaBeans的类，简化JavaBean的操作。
```
BeanWrapper instanceWrapper = null;
if (mbd.isSingleton()) {
  instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
}
if (instanceWrapper == null) {
  instanceWrapper = createBeanInstance(beanName, mbd, args);
}
final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
Class beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
```

#### 2. 合并BeanDefinition
调用`MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition`方法合并BeanDefinition // TODO 分析其具体作用
```
if (!mbd.postProcessed) {
  applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
  mbd.postProcessed = true;
}
```

#### 3.添加单例工厂，处理单例循环引用
这一步满足单例工厂，在工厂中获取当前已实例化的Bean，若出现循环依赖情况时，可直接从当前单例工厂中获取Bean，虽然这时的Bean还未进行初始化工作，但最终都会完成初始化，引用到正常的Bean。
```
boolean earlySingletonExposure = (mbd.isSingleton() 
  && this.allowCircularReferences 
  && isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
  addSingletonFactory(beanName, new ObjectFactory<Object>() {
    public Object getObject() throws BeansException {
      return getEarlyBeanReference(beanName, mbd, bean);
    }
  });
}
```

#### 4. 设置属性
这一步，设置Bean的属性，依赖注入等，注入是使用`InstantiationAwareBeanPostProcessor.postProcessPropertyValues`方法实现的。
```
populateBean(beanName, mbd, instanceWrapper);
```

#### 5. 初始化Bean
到这里后，Bean基本上已经完成装配，但还有一些初始化工作未完成。
```
exposedObject = initializeBean(beanName, exposedObject, mbd);
```
初始化工作主要有如下几个方面：  
1) 若Bean是BeanNameAware，BeanClassLoaderAware，BeanFactoryAware的实现，则调用相应aware方法注入上下文；  
2) 调用后处理器初始化前方法`BeanPostProcessor.postProcessBeforeInitialization`对Bean进行处理；  
3) 调用初始化方法，实现InitializingBean接口时调用`InitializingBean.afterPropertiesSet`，指定init-method时调用指定方法；  
4) 调用后处理器初始化后方法`BeanPostProcessor.postProcessAfterInitialization`（*Spring的Aop是在这个处理方法中实现的*）

#### 6. 注册销毁方法
注册销毁方法，当Spring容器销毁时，会调用销毁方法。
```
registerDisposableBeanIfNecessary(beanName, bean, mbd);
```
  
&#8195;&#8195;完成以上步骤后，Bean的创建工作基本上已经完成，可交由应用程序使用。特别地，在Bean初始化过程中，有Aware接口，BeanPostProcessor Bean创建后置处理器，InitializingBean初始化接口等，通过这些接口，可按需对Bean进行一些增强工作，对Spring功能进行扩展。

## 总结
&#8195;&#8195;在这一节中，从源码级学习了Spring获取和创建Bean的过程，大概如下：
> 1. 调用`DefaultSingletonBeanRegistry.getSingleton(java.lang.String, boolean)`方法中获取已创建的单例Bean对象，若获取到，直接返回；
> 2. 调用`InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation`实例化对象，若成功，直接返回；
> 3. 调用反射方法进行实例化对象；  
> 4. 设置Bean的属性及依赖注入；  
> 5. 调用Aware相关方法设置上下文；  
> 6. 调用`BeanPostProcessor.postProcessBeforeInitialization`对Bean进行加工；
> 7. 调用Bean初始化方法；
> 8. `BeanPostProcessor.postProcessAfterInitialization`对Bean进行后置增强；
> 9. 若是单例Bean，还会添加到DefaultSingletonBeanRegistry中。

&#8195;&#8195;以上是简单描述了Bean创建过程的部分流程，在Spring Bean的创建过程中，还有许多细节流程，比如父子容器Bean获取等没有讲述。此处，Spring还使用一些很巧妙的设计，来完成特定的工作，还需要细细品尝！  
&#8195;&#8195;在Bean创建过程中，有这样几个相关的接口/类：
> 1. **FactoryBean**：创建Bean的工厂，Spring最终返回的Bean是调用`FactoryBean.getObject`的对象；
> 2. **DefaultSingletonBeanRegistry**：Spring注册和获取单例Bean的类，实现了`SingletonBeanRegistry`接口，创建后的单例Bean都由该类进行管理；
> 3. **BeanWrapper**：用于简化JavaBean的操作；
> 4. **InstantiationAwareBeanPostProcessor**：可以对Bean的属性设置，依赖注入等功能；
> 5. **Aware**：Bean实现Aware接口后，在初始化时可设置相应的上下文信息；
> 6. **BeanPostProcessor**：Bean后置处理器，可在init方法调用前和后对Bean进行增强操作；
> 7. **InitializingBean**：实现该接口后，Spring会调用其`afterPropertiesSet`方法进行Bean的额外初始化，当然，也可以指定init-method；
> 8. **DisposableBean**：当非prototype类型的bean销毁时，会调用`destroy`，一般用于资源的释放操作。

&#8195;&#8195;本节学习的Spring Bean创建知识大概就这么多。从这节中了解到了Spring创建Bean的过程，并且结合前一节的[Spring从Xml加载Bean Definition的过程](http://blog.codedoge.com/2017/05/07/lesson-spring-source-ioc/)，可以了解到BeanFactory及其实现类为整个Spring Ioc搭建了一套功能强大，扩展性非常好的骨架，已经具备Ioc的基本功能，但是一些增强型的接口还需要手工进行注册（*比如BeanPostProcessor*），使用还稍许麻烦。在此之上，Spring还提供另一个容器接口`ApplicationContext`，在ApplicationContext中简化许多操作，使得Spring Ioc更加完整和强大，在后面，将会对`ApplicationContext`进行深入学习，了解`ApplicationContext`中是如何帮助我们简化/增强功能操作的。
