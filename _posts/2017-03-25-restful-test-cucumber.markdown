---
layout:     post
title:      "使用Cucumber测试REST API"
subtitle:   "学习BDD思想，对SpringMVC RESTful API做单元测试"
header-img: "img/in-post/20170325-restful-test-cucumber/header-bg.jpg"
header-mask: 0.3
catalog:    true
tags:
    - RESTful
    - SpringMVC
    - 单元测试
    - Cucumber
---

> 单元测试是大多数开发人员心里的痛，一方面，想保证代码开发质量，还得靠单元测试，但是传统Junit用例的编写复杂又会让开发人员望而却步。现在，BDD的出现，也许会让大家再次喜欢上编写单元测试。

## 传统Junit单元测试的不足
1. 把程序完全当成白盒，需要测试用例编写人员（一般是当前模块的开发人员）高度了解程序的实现逻辑；
2. 完全是白盒测试，容易形成对分支覆盖率、行覆盖率片面追求，但对测试质量不高；
3. 需要编写大量的测试代码（很多时候会比程序代码还多，测试越充分，测试代码越多）；
4. 测试用例不易理解，特别是对模块不熟悉以及其他没有编程基础的人；
5. 大面积重构需要变更的测试用例较多；
6. 无法站在用户（使用方）角度来对程序进行测试（这方面还得依靠集成测试来完成），使得“高质量”程序却可能无法满足用户需求，不符合需求和设计要求。

## Cucumber能解决什么问题
#### Cucumber介绍
&#8195;&#8195;[Cucumber](https://cucumber.io/)是一个BDD框架，使用更易于理解的自然语言来描述软件行为，可站在用户使用的角度对软件功能行为进行一系列描述。  
一个简单的Cucumber用例如下:
```
功能: 计算器
  场景: 两数相加
    假如我有一个计算器
    并且我向计算器输入100
    并且我向计算器输入20
    当我点击加号
    那么我应该看到结果130
```
&#8195;&#8195;是不是有种惊艳的感觉，没看错，测试用例确实能这样写。测试用例把程序当成黑盒，从用例使用角度对软件进行测试，是不是很完美呢？  
&#8195;&#8195;当然，介绍和学习[Cucumber](https://cucumber.io/)并不是本文重点，[Cucumber首页](https://cucumber.io/)有更多对其的介绍以及使用方法。

#### 如何解决传统单元测试的不足
1. 可把软件看做黑盒，测试用例编写人员无需了解其实现逻辑，同时，各利益相关人员都可参与到测试用例的编写中来；
2. 使用自然语言描述测试用例，更易于理解；
3. 测试基于各种功能场景，会追求场景的覆盖，而不会片面要求代码分支或代码行的覆盖率，使测试用例的编写以用户使用场景为核心，回归测试正常轨道；
4. 增加测试场景不会大量增加编码工作量，但会编写测试用例，但相对于以前的编码工作，工作量上会少很多；
5. 把软件看成黑盒，只需要关注程序的输入输出，在程序大量重构时，只要没有改变程序的接口定义时，测试用例无需重新编写；
6. 从用户使用角度来对软件行为进行描述，完全符合用户使用行为，只要用例场景符合用户使用行为且测试通过，则软件一定是符合用户需求，不会出现即使测试通过，软件也无法满足用户需求的问题。

## 传统RestAPI测试遇到的问题
&#8195;&#8195;传统的RestAPI测试都是外部测试（如：[Postman](https://www.getpostman.com/)），需要被测程序牌启动状态，并且无法绕过程序一些安全拦截（比如登录，验证码等），同时，因测试代码处理程序外部，无法对被测程序所依赖的第三方接口进行Mock。

## 使用Cucumber对SpringMVC RestAPI进行测试
#### Cucumber集成Spring
&#8195;&#8195;需要使用Cucumber对Spring程序进行测试，需要进行Cucumber与Spring的整合工作。集成cucumber与spring，除依赖普通的cucumber与spring-test包外，还需要依赖cucumber-spring包，若Maven依赖如下：
```
<dependency>
    <groupId>info.cukes</groupId>
    <artifactId>cucumber-spring</artifactId>
    <version>${cucumber.version}</version>
    <scope>test</scope>
</dependency>
```
&#8195;&#8195;Cucumber运行时会加载classpath下的cucumber.xml文件，这是一个Spring配置文件，cucumber会按照cucumber.xml中的配置加载spring上下文。配置文件和普通的spring项目一致，可适当去掉一些非核心功能，比如登录验证等。
#### REST测试步骤编写
&#8195;&#8195;需要通过Cucumber模拟向程序发送Http请求，并验证返回值。代码验证部分未贴出，过根据自己项目需要，进行不同的验证，建议对项目API进行整体设计，值请求和返回具有一定格式。
```
public class RestTestStep {

    @Autowired
    private WebApplicationContext wac;

    private MockMvc mockMvc;

    private String url;
    private String method;
    private Map<String, String> param;

    private MvcResult result;

    @Before
    public void setup() {
        this.mockMvc = webAppContextSetup(this.wac).build();
        param = new HashMap<String, String>();
    }

    @假如("^请求URL为:(.*)$")
    public void setUrl(String url) throws Exception {
        this.url = url;
    }

    @假如("^请求方法为:(.*)$")
    public void setMethod(String method) throws Exception {
        this.method = method;
    }

    @假如("^请求参数(.*)的值为:\"(.*)\"$")
    public void addParam(String name, String value) throws Exception {
        param.put(name, value);
    }

    @当("^发送请求$")
    public void doRequest() throws Exception {
        MockHttpServletRequestBuilder requestBuilder =
                MockMvcRequestBuilders.
                request(HttpMethod.valueOf(method.toUpperCase()), url);
        Set<Map.Entry<String, String>> entries = param.entrySet();
        for (Map.Entry<String, String> param : entries) {
            requestBuilder.param(param.getKey(), param.getValue());
        }
        result = mockMvc.perform(requestBuilder).andDo(print()).andReturn();
    }

    @那么("^响应状态码为:(\\d{3})")
    public void validateStatus(int expectedCode) {
        int status = result.getResponse().getStatus();
        Assert.assertEquals("状态码与预期不相同", expectedCode, status);
    }
}
```
#### 加载Spring WEB上下文
&#8195;&#8195;上面的步骤编写中，需要注入`WebApplicationContext`Bean,但cucumber默认加载的上下文为`ClassPathXmlApplicationContext`，所以我们需要手动创建`WebApplicationContext`（_注：可能是我依赖的Spring版本过低，暂未发现其他简便方法，若您还知道其他方法，可否在下面回复告诉一下，感激不尽_）  
我实现代码如下，使用工厂Bean初始化`XmlWebApplicationContext`：
```
@Configuration
public class WebApplicationContextConfiguration implements ApplicationContextAware {
    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext)  {
        this.applicationContext = applicationContext;
    }

    @Bean
    public WebApplicationContext getBean() {
        MockServletContext servletContext = new MockServletContext();
        servletContext.setMajorVersion(2);
        servletContext.setMinorVersion(4);
        servletContext.setContextPath("");
        XmlWebApplicationContext webApplicationContext = 
                                              new XmlWebApplicationContext();
        webApplicationContext.setConfigLocation("classpath:application-servlet.xml");
        servletContext.registerNamedDispatcher("default",
                                               new MockRequestDispatcher("default"));
        webApplicationContext.setServletContext(servletContext);
        webApplicationContext.setParent(applicationContext);
        webApplicationContext.refresh();
        return webApplicationContext;
    }
}
```
#### 编写Rest测试用例
&#8195;&#8195;完成以上整合工作后，就可以使用编写cucumber场景来对我们的RESTful API进行测试。我自己的一个简单例子片断：
````
# language: zh-CN
功能:修改用户信息

  ## 正常流程测试
  场景:测试修改用户正常流程
    ## URL 信息
    假如请求URL为:/user/123456
    而且请求方法为:POST

    而且请求参数userName的值为:"testUser"
    而且请求参数phone的值为:"13666666666"
    而且请求参数email的值为:"testUser@codedoge.com"

    # 发送请求
    当发送请求

    # 响应结果验证
    那么响应状态码为:200
    # 客户端参数验证异常
    而且返回中userId的值为:"123456"
    而且返回中status的值为:"SUCCESS"

    ## 获取用户信息，看是否修改成功
    假如请求URL为:/user/123456
    而且请求方法为:GET

    # 发送请求
    当发送请求

    # 响应结果验证
    那么响应状态码为:200
    而且返回中userId的值为:"123456"
    而且返回中userName的值为:"testUser"
    而且返回中phone的值为:"13666666666"
    而且返回中email的值为:"testUser@codedoge.com"
````
&#8195;&#8195;因为测试并不是发送真正的Http请求，所以不需要启动服务器。同时，可在程序中对各后台数据进行Mock。

## 总结
&#8195;&#8195;结合BDD思想，在用户的角度，对程序进行测试，模拟用户操作向程序发送对应请求，以验证程序是否符合设计要求。同时，只需要关注程序边界上，其他干扰信息较少，对开发人员设计出高质量API留出更多时间以思考方式。  
&#8195;&#8195;但是该测试方法并非是完全替代传统的Junit单元测试，在一些算法，工具类，或需要更细粒度测试的时候，还是Junit的支持范围。两者组合使用，效果更佳哦！！  
  
&#8195;&#8195;本人刚开始学习Cucumber，文章也写得较少，若有错误或不足之处，还请指正，万分感谢！




