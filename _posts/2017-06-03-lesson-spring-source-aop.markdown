---
layout:     post
title:      "学习笔记 - Spring源码学习五"
subtitle:   "Spring AOP原理及初始化"
header-img: "img/in-post/spring-lesson/spring-logo.jpg"
header-mask: 0.2
catalog:    true
tags:
    - Spring
    - Aop
---

> &#8195;&#8195;前几篇笔记已经学习了Spring Ioc的内容，学习到Spring Ioc框架中的主要核心BeanFactory、ApplicationContext的组成和初始化过程，以及在Spring是如何进行Bean对象的管理。今天，将着重学习Spring Aop的原理以及初始化过程，加深对其了解。

## Spring Aop原理及相关概念
&#8195;&#8195;在Spring中，Aop是非常重要的一个功能。在日志记录、事务管理、缓存等等方面都有较成熟的应用。通过动态代理(jdk,cglib)实现，在运行时进行功能增强。在容器中存在类型为`AbstractAutoProxyCreator`的Bean后置处理器(*[Bean初始化过程](http://blog.codedoge.com/2017/05/10/lesson-spring-source-beaninit/)中有描述*)，可以使得在getBean时生成代理对象，置入相应的Aop代码。在Spring Aop中有几个相关的重要概念：
- **Pointcut**:切点，指定需要代理的目标，通过（包、类、方法、注解等方式配置）
- **Advice**:通知，在执行代理目标方法前、后或者异常时执行，类型有Before(在目标方法执行前执行)、Around(在目标方法执行前、后执行，环绕通知)、After(在目标方法执行后执行)、AfterReturning(在目标方法执行后执行-正常返回)、AfterThrowing(在目标方法异常后执行)，*要深刻理解这几种方式的区别，可参考大神博客－[Aop before after afterReturn afterThrowing around 的原理](http://www.cnblogs.com/f1194361820/p/5513106.html)*

&#8195;&#8195;在Spring Aop初始化过程中，将我们配置的Pointcut及Advice生成相应的MethodInterceptor，通过cglib或jdk生成动态代理对象，然后在目标调用的不同阶段调用不同的Advice方法。

## Spring Aop源码分析
&#8195;&#8195;Spring Aop的核心原理其实就是动态代理，从这个角度考虑可以分为生成代理对象以及初始化(准备数据阶段)。在初始化阶段将我们定义的各种Pointcut和Advice生成在生成代理对象过程中需要的数据信息，而在生成代理对象过程时，将使用cglib或jdk动态代理方式生成动态代理对象，具体哪些类、哪些方法需要进行代理，这些数据在初始化阶段就准备好了。
#### 生成代理对象
&#8195;&#8195;为什么先于**初始化**阶段来学习**生成代理对象**过程？因为初始化过程中的数据都是为生成代理对象过程准备的，在理解了生成代理对象过程中需要些什么样的数据后，再回去学习初始化过程，会有更清晰的认识。  
&#8195;&#8195;在容器中存在`AbstractAutoProxyCreator`类型的Bean后置处理器，在初始化bean时会调用其`postProcessAfterInitialization`方法生成代理对象。
```
public Object postProcessAfterInitialization(Object bean, String beanName) {
  if (bean != null) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    if (!this.earlyProxyReferences.containsKey(cacheKey)) {
      return wrapIfNecessary(bean, beanName, cacheKey);
    }
  }
  return bean;
}
```
&#8195;&#8195;`postProcessAfterInitialization`方法中，调用`wrapIfNecessary`方法对bean进行处理，如果有必要的会对bean进行代理并返回代理后的对象。
```
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
  // Create proxy if we have advice.
  Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
  if (specificInterceptors != null) {
    Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    return proxy;
  }
  return bean;
}
```
&#8195;&#8195;首先通过class和beaNname获取代理需要的advice，若不空，则表示当前bean不需要被包装代理，否则调用createProxy生成代理对象。`getAdvicesAndAdvisorsForBean`方法由子类具体实现。
```
protected Object createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {
  ProxyFactory proxyFactory = new ProxyFactory();
  // Copy our properties (proxyTargetClass etc) inherited from ProxyConfig.
  proxyFactory.copyFrom(this);
  if (!shouldProxyTargetClass(beanClass, beanName)) {
    // Must allow for introductions; can't just set interfaces to
    // the target's interfaces only.
    Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, this.proxyClassLoader);
    for (Class<?> targetInterface : targetInterfaces) {
      proxyFactory.addInterface(targetInterface);
    }
  }
  Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
  for (Advisor advisor : advisors) {
    proxyFactory.addAdvisor(advisor);
  }
  proxyFactory.setTargetSource(targetSource);
  customizeProxyFactory(proxyFactory);
  proxyFactory.setFrozen(this.freezeProxy);
  if (advisorsPreFiltered()) {
    proxyFactory.setPreFiltered(true);
  }
  return proxyFactory.getProxy(this.proxyClassLoader);
}
```
&#8195;&#8195;createProxy会调用proxyFactory的getProxy来生成代理对象，createProxy方法中大部分工作都是在构造ProxyFactory，设置代理所需要信息。比如需要代理的对象、接口、代理通知以及其他信息等，最后调用getProxy方法生成最终代理后的对象。
```
public Object getProxy(ClassLoader classLoader) {
  return createAopProxy().getProxy(classLoader);
}
```
&#8195;&#8195;最终会通过AopProxy.getProxy方法生成代理对象。createAopProxy方法是创建AopProxy对象，以选择是通过cglib还是jdk的方式去实现动态代理。接下来我们分析jdk的方式，cglib代理的原理也基本相同。AopProxy的jdk方式实现类是JdkDynamicAopProxy。
![Spring JdkDynamicAopProxy类图](/img/in-post/spring-lesson/aop/class-JdkDynamicAopProxy.png)
&#8195;&#8195;JdkDynamicAopProxy实现了AopProxy接口，调用其getProxy可获取动态代理后的对象，同时，也实现了InvocationHandler接口，这是jdk动态代理中的一个重要接口，在代理对象执行方法时，会执行其invoke方法。
```
public Object getProxy(ClassLoader classLoader) {
  Class[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
  findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
  return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```
&#8195;&#8195;getProxy方法中其实就是简单的jdk代理过程。在代理对象方法执行时，会执行invoker方法，我们再看一下invoker方法的实现是怎样的。
```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  TargetSource targetSource = this.advised.targetSource;
  // 对equals和hashCode等方法单独处理
  if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
    return equals(args[0]);
  }
  if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
    return hashCode();
  }
  if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
      method.getDeclaringClass().isAssignableFrom(Advised.class)) {
    return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
  }

  Object target = targetSource.getTarget();
  Class targetClass = null;
  if (target != null) {
    targetClass = target.getClass();
  }

  List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

  Object retVal;
  if (chain.isEmpty()) {
  　// 代理通知为空时，直接反射调用目标方法
    retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
  }
  else {
    // 其他情况通过ReflectiveMethodInvocation.proceed调用。
    retVal = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain).proceed();
  }
  return retVal;
}
```
&#8195;&#8195;在ReflectiveMethodInvocation.proceed中其实就是在不同阶段调用相应的MethodInterceptor.invoker方法，在真正需要的时候调用原始方法。  

#### 初始化
&#8195;&#8195;初始化过程就是为生成代理时使用的数据，其实最主要的也是Advice及Pointcut，那我们就直及主题，来从源码级别来看一下Spring Aop是如何初始化的Advice及Pointcut的。  
&#8195;&#8195;在AbstractAutoProxyCreator.wrapIfNecessary方法中调用getAdvicesAndAdvisorsForBean方法获取Advisor。其实在这里会产生分化了，首先会获取容器中注册的所有Advisor类型的advisor，若配置了注解aop，还需要获取所有被Aspect注解的Bean，然后生成相应的Advisor对象。

## 总结
&#8195;&#8195;Spring Aop使用JDK动态代理或cglib生成字节码来对Bean进行增强，这个过程在Bean初始化时调用AbstractAutoProxyCreator类型的Bean后置处理器实现，会从容器中所有Advisor类型的Bean以及所有以Acepect注解的Bean中获得相应的切点与通知信息，相应地对Bean对象进行增强。若我们要使用Aop，一方面可以使用spring提供的一系列xml配置向容器中提供Advisor Bean，或者使用Aspect注解Bean均可
