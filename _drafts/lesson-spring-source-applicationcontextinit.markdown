---
layout:     post
title:      "学习笔记 - Spring源码学习四"
subtitle:   "ApplicationContext初始化过程"
header-img: "img/in-post/spring-lesson/spring-logo.jpg"
header-mask: 0.2
catalog:    true
tags:
    - Spring
---

> 前两篇笔记学习了[Spring从Xml加载Bean Definition的过程](http://blog.codedoge.com/2017/05/07/lesson-spring-source-ioc/)以及[Bean初始化过程](http://blog.codedoge.com/2017/05/10/lesson-spring-source-beaninit/)，了解到了在AbstractBeanFactory及其子类中定义了Spring Ioc的骨架，已经可以使用使用Ico的功能，但是某些特定的Bean还需要手工进行初始化，而ApplicationContext。

## ApplicationContext
先上一张大图。
![](/img/in-post/spring-lesson/applicationcontextinit/class-ApplicationContext-all.png)