---
layout:     post
title:      "学习笔记 - Spring源码学习二"
subtitle:   "从XmlBeanDefinitionReader实现学习从Xml加载Bean的过程"
header-img: "img/in-post/spring-lesson/spring-logo.jpg"
header-mask: 0.2
catalog:    true
tags:
    - Spring
    - BeanDefinitionReader
---
> 上篇笔记学习了[Spring Ioc容器的结构体系](http://blog.codedoge.com/2017/05/07/lesson-spring-source-ioc/)，了解了Spring中BeanFactory、Registry、BeanDefinitionReader的作用。今天，将学习了解BeanDefinitionReader是如何将bean定义从资源文件中加载到容器中的。

## BeanDefinitionReader分析
&#8195;&#8195;在接口BeanDefinitionReader上有这样一段注释。
> Simple interface for bean definition readers. Specifies load methods with Resource and String location parameters.Concrete bean definition readers can of course add additional load and register methods for bean definitions, specific to their bean definition format. Note that a bean definition reader does not have to implement this interface. It only serves as suggestion for bean definition readers that want to follow standard naming conventions.

&#8195;&#8195;大意是“ 该接口定义了从Resource和location加载bean定义的方法，具体的Reader可以添加额外的加载和注册方法。***需要注意的是，bean definition Reader不必实现此接口。 它仅作为要遵循标准命名约定的bean定义阅读器的建议*** ”(*注：Spring2.5后提供的从注解加载Bean就没有实现该接口*)，实现该Reader的有PropertiesBeanDefinitionReader和XmlBeanDefinitionReader。　　
&#8195;&#8195;现在，将通过源码简单学习XmlBeanDefinitionReade的原理，理解Spring从Xml文件中加载bean的过程。

#### 类图及方法
&#8195;&#8195;先看看BeanDefinitionReader中定义的方法
![Spring XmlBeanFactory类图](/img/in-post/spring-lesson/beandefinitionreader/class-BeanDefinitionReader.png)
&#8195;&#8195;从类图可以看到，BeanDefinitionReader定义了4个重载的loadBeanDefinitons方法，用于从不同的资源、位置加载bean的定义，还提供了获取Bean容器Registry，以及bean name生成器、资源加载器、类加载器等方法。

## XmlBeanDefinitionReader分析
&#8195;&#8195;XmlBeanDefinitionReader是BeanDefinitionReader的实现类，用于从xml文件中加载bean定义。
```
public XmlBeanDefinitionReader(BeanDefinitionRegistry registry) {
　　super(registry);
}
public int loadBeanDefinitions(Resource resource) {
  return loadBeanDefinitions(new EncodedResource(resource));
}

public int loadBeanDefinitions(EncodedResource encodedResource) {
  InputStream inputStream = encodedResource.getResource().getInputStream();
  try {
    InputSource inputSource = new InputSource(inputStream);
    if (encodedResource.getEncoding() != null) {
      inputSource.setEncoding(encodedResource.getEncoding());
    }
    return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
  }finally {
    inputStream.close();
  }
}
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) {
  int validationMode = getValidationModeForResource(resource);
  Document doc = this.documentLoader.loadDocument(
    inputSource, getEntityResolver(), this.errorHandler, validationMode,
    isNamespaceAware());
  return registerBeanDefinitions(doc, resource);
}
public int registerBeanDefinitions(Document doc, Resource resource) {
  BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
  documentReader.setEnvironment(this.getEnvironment());
  int countBefore = getRegistry().getBeanDefinitionCount();
  documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
  return getRegistry().getBeanDefinitionCount() - countBefore;
}
```
&#8195;&#8195;上面是`XmlBeanDefinitionReader.loadBeanDefinitions(Resource)`的部分相关源码。从源码中可以看出，XmlBeanDefinitionReader在初始化的时候需要依赖BeanDefinitionRegistry，从xml文件中解析出的BeanDefinition就注册到该Registry中。在解析bean definition方面，资源文件中bean定义具体是通过`BeanDefinitionDocumentReader.registerBeanDefinitions`方法实现的，其文档上是这样描述的。*Read bean definitions from the given DOM document and register them with the registry in the given reader context.*

## BeanDefinitionDocumentReader分析
```
public interface BeanDefinitionDocumentReader {

  void setEnvironment(Environment environment);

  void registerBeanDefinitions(Document doc, XmlReaderContext readerContext)
      throws BeanDefinitionStoreException;

}
```
这是BeanDefinitionDocumentReader的源码定义，很简单，只有两个方法。从名字上可以看出，`setEnvironment`用于设置环境变量，*还记得profile吗？就是靠这来区别的*；`registerBeanDefinitions`就是解析注册具体的bean definition了，第一个参数是xml文档，第二个参数上解析的上下文，包含了解析上下文所需要的资源，最重要的是包含了当前reader，可从中获取registry。

## DefaultBeanDefinitionDocumentReader分析
&#8195;&#8195;DefaultBeanDefinitionDocumentReader是BeanDefinitionDocumentReader的默认实现，XmlBeanDefintion默认也是通过DefaultBeanDefinitionDocumentReader来解析bean definition的。
```
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
  this.readerContext = readerContext;
  Element root = doc.getDocumentElement();
  doRegisterBeanDefinitions(root);
}
protected BeanDefinitionParserDelegate createDelegate(XmlReaderContext readerContext, Element root, BeanDefinitionParserDelegate parentDelegate) {
  //　省略部分代码
  delegate = new BeanDefinitionParserDelegate(readerContext, this.environment);
  delegate.initDefaults(root, parentDelegate);
  return delegate;
}
protected void doRegisterBeanDefinitions(Element root) {
  // 省略profile判断
  BeanDefinitionParserDelegate parent = this.delegate;
  this.delegate = createDelegate(this.readerContext, root, parent);

  preProcessXml(root);
  parseBeanDefinitions(root, this.delegate);
  postProcessXml(root);

  this.delegate = parent;
}

protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
  if (delegate.isDefaultNamespace(root)) {
    NodeList nl = root.getChildNodes();
    for (int i = 0; i < nl.getLength(); i++) {
      Node node = nl.item(i);
      if (node instanceof Element) {
        Element ele = (Element) node;
        if (delegate.isDefaultNamespace(ele)) {
          parseDefaultElement(ele, delegate);
        } else {
          delegate.parseCustomElement(ele);
        }
      }
    }
  } else {
    delegate.parseCustomElement(root);
  }
}
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
  if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
    importBeanDefinitionResource(ele);
  } else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
    processAliasRegistration(ele);
  } else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
    processBeanDefinition(ele, delegate);
  } else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
    doRegisterBeanDefinitions(ele);
  }
}
```
&#8195;&#8195;从源码上看，最终的解析会委托给BeanDefinitionParserDelegate，在委托前。需要做这样的处理。
1. 如果根节点是非默认命名空间节点(*`BEANS_NAMESPACE_URI.equals(namespaceUri)`，即beans*)，则委托给`delegate.parseCustomElement`解析
2. 否则获取子节点，如果子节点是节点是非默认命名空间节点，同样委托给`delegate.parseCustomElement`解析，进入`parseDefaultElement`方法处理。
3. 在`parseDefaultElement`方法中，会针对不同节点名称做不同的处理，如import,alias,bean,beans等。

#### import节点解析
&#8195;&#8195;import节点就是把解析resource属性，并再次调用上下文中的BeanDefinitonReader进行解析。只是该方法中对绝对路径等不同情况做了处理，可以想像spring开发者的严谨。
```
protected void importBeanDefinitionResource(Element ele) {
  String location = ele.getAttribute("resource");
  Set<Resource> actualResources = new LinkedHashSet<Resource>(4);

  boolean absoluteLocation  = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
  if (absoluteLocation) {
    int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
  }
  else {
    int importCount;
    Resource relativeResource = getReaderContext().getResource().createRelative(location);
    if (relativeResource.exists()) {
      importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
      actualResources.add(relativeResource);
    }
    else {
      String baseLocation = getReaderContext().getResource().getURL().toString();
      importCount = getReaderContext().getReader().loadBeanDefinitions(
          StringUtils.applyRelativePath(baseLocation, location), actualResources);
    }
  }
  Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
  getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}
```
#### alias节点解析
&#8195;&#8195;解析就更简单了，获取值，并注册到上下文中的registry中即可。
```
protected void processAliasRegistration(Element ele) {
  String name = ele.getAttribute("name");
  String alias = ele.getAttribute("alias");
  getReaderContext().getRegistry().registerAlias(name, alias);
  getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
}
```

#### bean节点解析
&#8195;&#8195;Bean节点的解析委托给了BeanDefinitionParserDelegate，然后再注册到上下文的registry中。BeanDefinitionParserDelegate构造BeanDefiniton就是简单的get set，没有太多的逻辑，这里暂时不做分析。
```
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
  BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
  if (bdHolder != null) {
    bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
    BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
    getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
  }
}
```

#### beans节点解析
&#8195;&#8195;在解析到beans节点时，再次递归调用了`doRegisterBeanDefinitions`方法，现在回过头去看一下`doRegisterBeanDefinitions`的源码，其中两句就是用来处理这种有多层级beans标签的，子beans节点可以使用父beans节点的一些属性。
```
BeanDefinitionParserDelegate parent = this.delegate;
this.delegate = createDelegate(this.readerContext, root, parent);
```
BeanDefinitionParserDelegate初始化默认值方法
```
populateDefaults(this.defaults, (parent != null ? parent.defaults : null), root);
```

#### 非默认命名空间节点解析
&#8195;&#8195;非默认命名空间节点解析会委托给`delegate.parseCustomElement`，那么在parseCustomElement中又是怎么找到解析方法的呢！
```
public BeanDefinition parseCustomElement(Element ele) {
  return parseCustomElement(ele, null);
}

public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
  String namespaceUri = getNamespaceURI(ele);
  NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
  return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```
&#8195;&#8195;从代码中可以看到，spring会根据节点的命名空间找到对应的NamespaceHandler来解析。以下是如何寻找对应NamespaceHandler的源码：
```
public NamespaceHandler resolve(String namespaceUri) {
  Map<String, Object> handlerMappings = getHandlerMappings();
  Object handlerOrClassName = handlerMappings.get(namespaceUri);
  if (handlerOrClassName == null) {
    return null;
  } else if (handlerOrClassName instanceof NamespaceHandler) {
    return (NamespaceHandler) handlerOrClassName;
  } else {
    String className = (String) handlerOrClassName;
    Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
    NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
    namespaceHandler.init();
    handlerMappings.put(namespaceUri, namespaceHandler);
    return namespaceHandler;
  }
}

private Map<String, Object> getHandlerMappings() {
  if (this.handlerMappings == null) {
    synchronized (this) {
      if (this.handlerMappings == null) {
        Properties mappings =  PropertiesLoaderUtils.loadAllProperties("META-INF/spring.handlers", this.classLoader);
        Map<String, Object> handlerMappings = new ConcurrentHashMap<String, Object>(mappings.size());
        CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
        this.handlerMappings = handlerMappings;
      }
    }
  }
  return this.handlerMappings;
}
```
&#8195;&#8195;从源码中可以看出，第一次加载时，spring默认会从META-INF/spring.handlers文件中加载nameSpaceUri与NamespaceHandler class的映射，然后再实例化、调用init方法初始化，最后返回给调用者使用。spring使用了一个很巧妙的设计，在同一个map里存储class name和handler对象。  
&#8195;&#8195;我们具体在使用的时候更多是继承NamespaceHandlerSupport，然后在init方法中初始化node name与BeanDefinitionParser映射，BeanDefinitionParser再做具体的解析工作。

## 总结
&#8195;&#8195;至此，Spring从Xml文件中加载Bean也了解得7788了，从源码上了解了整个过程以及xml文件的结构。
> 记了大好一篇口水账，需要好好进行总结，日后也好作参考。

1. Spring通过BeanDefinitionReader来加载资源文件中的bean definition到Ioc容器中;
2. XmlBeanDefinitionReader将资源文件转换为xml dom对象，然后交给BeanDefinitionDocumentReader做具体的解析工作；
3. 在DefaultBeanDefinitionDocumentReader中，会根据根节点的命名空间做不同解析，若在非beans空间下，会委托给BeanDefinitionParserDelegate.parseCustomElement解析，若在beans空间，会根据不同的bean name做不同的处理，遇到名为import的标签会再次调用BeanDefinitionReader解析，遇到alias会解析别名到registry中，遇到bean会调用delegate.parseBeanDefinitionElement方法解析具体的bean definition，而遇到beans时会递归调用doRegisterBeanDefinitions进行解析；
4. 在其他命名空间下的节点，Spring会查找位于classpath:META-INF/spring.handlers，从中找到节点命名空间与NamespaceHandler的映射关系。在初始化时会调用NamespaceHandler的init方法进行初始化工作。
5. 我们具体在使用的时候更多是继承NamespaceHandlerSupport，然后在init方法中初始化node name与BeanDefinitionParser映射，BeanDefinitionParser再做具体的bean解析工作（*大部分的spring其他模块就是通过这种方式实现的*）
6. BeanDefinitionParser.parse解析bean时，可以通过ParserContext参数获取上下文信息，包括BeanDefinitionReader,BeanDefinitonRegistry等。
7. 虽然Spring经常使用BeanDefinitionReader来加载bean definiton，但这不是强制要求，比如Spring注解定义Bean就不是通过BeanDefinitionReader来实现。