---
layout: default
title: "Spring从XML加载BeanDefinition源码分析"
data: 2022-09-18 16:58:00 -0000
category: spring
---

本文分析Spring是如何将我们定义的Bean加载到Spring容器中。

Spring的版本是 5.2.6.RELEASE
Dubbo版本是 2.6.2

本文是Spring IOC 初始化流程的一部分，回到初始化过程中的 refreshBeanFacotry 方法。

本文分析从XML中加的源码逻辑。

```java
// AbstractRefreshableApplicationContext.class 121
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        // 加载BeanDefinition。我们使用Spring主要从XML和注解两种形式加载bean。
        loadBeanDefinitions(beanFactory);
        this.beanFactory = beanFactory;
    }
    catch (IOException ex) {...}
}
```

创建一个Reader，也就是读取器，它的作用是将我们配置的XML形式的bean加载到Spring中。

```java
// AbstractXmlApplicationContext.class 81
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // 创建一个Reader，就是它把我们配置在XML中的bean加载到Spring中
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
    initBeanDefinitionReader(beanDefinitionReader);
    
    // 开始加载
    loadBeanDefinitions(beanDefinitionReader);
}
```

**从Resource资源和ClassPath类型的资源加载。**

ClassPath这种加载方式先会把我们的配置加载成Resource，然后依然使用Resource的方式去加载。

```java
// AbstractXmlApplicationContext.class 121
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    // 从Resource加载
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    // 从configLocations加载，本质也是先把configLocations转成Resource，再调用上面这个加载方法
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        reader.loadBeanDefinitions(configLocations);
    }
}

public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int count = 0;
    // 有多个Resource则循环加载
    for (Resource resource : resources) {
        count += loadBeanDefinitions(resource);
    }
    return count;
}
```

使用set记录已加载的资源，避免重复加载。

**将资源转成输入流**的形式，作为下一个方法的入参。

```java
// XmlBeanDefinitionReader.class 320
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    
    // 使用set，避免重复加载。已加载Resource的放到set中
    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    if (!currentResources.add(encodedResource)) {...}
    // 将资源转成输入流的形式
    try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
        InputSource inputSource = new InputSource(inputStream);
        if (encodedResource.getEncoding() != null) {
            inputSource.setEncoding(encodedResource.getEncoding());
        }
        // 从这里进去
        return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
    }
    catch (IOException ex) {...}
    finally {...}
}
```

这是Reader的核心方法了，这里做两个事：**将输入流加载成Document**，**将Document注册到Spring中**。

```java
// XmlBeanDefinitionReader.class 386
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
        throws BeanDefinitionStoreException {

    try {
        Document doc = doLoadDocument(inputSource, resource);
        int count = registerBeanDefinitions(doc, resource);
        if (logger.isDebugEnabled()) {
            logger.debug("Loaded " + count + " bean definitions from " + resource);
        }
        return count;
    }
    catch (BeanDefinitionStoreException ex) {...}
    catch (SAXParseException ex) {...}
    catch (SAXException ex) {...}
    catch (ParserConfigurationException ex) {...}
    catch (IOException ex) {...}
    catch (Throwable ex) {...}
}
```

**Document，Element，Node之间的关系**
> Node指的是XML中的一个节点，这是w3c定义的。
> XML中的Node有很多种类型，其中一种是Element，我理解Element就是XML中的标签。
> XML中的Node还一种类型是Attribute，它其实就是Element的一个属性，<bean id="123"/>，例如id就是一个Attribute。
> Document是XML文件对应的一个实体，它是一个树状结构。
> 它有一个根节点是Element类型的，每个Element包含一个子Element列表和Attribute列表。

**阶段性总结**
到这里我们总结一下前面的步骤。
1. 创建Reader用于将我们配置的Bean加载到Spring中
2. 支持从Resource和ClassPath中加载
3. 将Resource转成输入流
4. 将输入流转成Document
5. 将Document加载到Spring

接下来的步骤是：创建一个文档读取器，读取文档中的bean并注册，并返回本次注册的bean数量

```java
// XmlBeanDefinitionReader.class 508
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    // 看这里
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}

// DefaultBeanDefinitionDocumentReader.class 94
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    // 获取文档的根节点去解析
    doRegisterBeanDefinitions(doc.getDocumentElement());
}
```

**解析默认标签（import，alias，bean，beans）和用户自定义标签**

```java
// DefaultBeanDefinitionDocumentReader.class 121
protected void doRegisterBeanDefinitions(Element root) {

    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);
    // 如果当前是默认标签，验证是否合法
    if (this.delegate.isDefaultNamespace(root)) {
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                    profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                            "] not matching: " + getReaderContext().getResource());
                }
                return;
            }
        }
    }
    // 钩子方法
    preProcessXml(root);
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root);

    this.delegate = parent;
}

// DefaultBeanDefinitionDocumentReader.class 168
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    // 如果是头部标签则获取子节点然后遍历加载。
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                // 迭代判断是默认标签还是用户自定义标签
                if (delegate.isDefaultNamespace(ele)) {
                    parseDefaultElement(ele, delegate);
                }
                else {
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        // 到了这里说明是用户自定义的标签，那么得用用户自定义的解析器来解析。例如DubboNameSpaceHandler
        delegate.parseCustomElement(root);
    }
}
```

先关注bean标签如何解析，后续关注用户自定义标签如何解析

#### 解析Spring默认标签

如果是Spring默认一级标签，调用对应的解析方法。

注意看beans标签的解析方法，其实又走到我们前面解析的方法了，因为XML的最外层的标签就是beans

```java
// DefaultBeanDefinitionDocumentReader.class 121
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        importBeanDefinitionResource(ele);
    }
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        processAliasRegistration(ele);
    }
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        // 关注这个解析
        processBeanDefinition(ele, delegate);
    }
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        // 递归
        doRegisterBeanDefinitions(ele);
    }
}
```

一个bean标签的解析无非三个事:
1. 将Element解析为BeanDefinitionHolder
2. 修饰BeanDefinitionHolder
3. 将holder注册到Spring

```java
// DefaultBeanDefinitionDocumentReader.class 305
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 将Element解析为Holder
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 不知道
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 将Holder注册到Spring
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {...}
        // 发送bean注册成功的事件
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```
**阶段性总结**

如果是默认标签接下来怎么做？
1. 根据标签类型调用对应解析方法
2. 如果是beans标签无非就是继续递归
3. 如果是bean标签。
4. 那么先将Element解析成BeanDefinitionHolder
5. 修饰BeanDefinitionHolder
6. 将BeanDefinitionHolder注册到Spring。

BeanDefinitionHolder和BeanDefinition有什么区别？
> 包含关系。BeanDefinitionHolder包含一个BeanDefinition、beanName以及别名列表，这些数据后续都需要注册到Spring中。
> 看下BeanDefinitionHolder类就很清楚了。

那么将Element解析成BeanDefinitionHolder这个操作具体是怎么做的呢？
* 解析id元素
* 解析name元素
* 通过id和name确定这个bean的唯一标识也就是:beanName
* 解析其余的Attribute属性
* 解析其余ChildNode属性。

Attribute属性和ChildNode有什么区别？
> /<bean id="20162501" name="欧阳">
>    /<meta key="beanName" value="beanDefinition"/>
>    /<qualifier value="a" type="java.lang.Integer"/>
> /</bean>
> 就拿这个bean标签举例，attribute就是标签内部的元素也就是id和name。
> ChildNode子节点就是meta节点和qualifier节点，它是被bean包含的结点。

```java
// BeanDefinitionParserDelegate.class 404
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    return parseBeanDefinitionElement(ele, null);
}

// BeanDefinitionParserDelegate.class 414
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
    // <bean id="20162501" name="欧阳,子明"/>
    // 这里是获取id也就是20162501
    String id = ele.getAttribute(ID_ATTRIBUTE);
    // 这里是获取name属性，也就是欧阳,子明
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

    // 将name属性解析为别名。
    List<String> aliases = new ArrayList<>();
    if (StringUtils.hasLength(nameAttr)) {
        // 根据,或者;分割name属性，然后添加到别名数组中。分割之后为 ["欧阳","子明"]
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }

    String beanName = id;
    // 如果没有设置id，将name的第一个名称设置为id。也就是欧阳
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        beanName = aliases.remove(0);
    }

    // 判断beanName和aliases中的别名是不是全局唯一。如果不是全局唯一则抛错。
    if (containingBean == null) {
        checkNameUniqueness(beanName, aliases, ele);
    }
    
    // 将Element解析为beanDefinition。基本思路就是获取Element中对应的属性，然后将属性set到beanDefinition中。
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
    if (beanDefinition != null) {
        // 如果这个bean既没有定义id也没有定义name，那就将类名作为beanName。
        if (!StringUtils.hasText(beanName)) {}
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }

    return null;
}
```

怎么装饰BeanDefinitionHolder的呢？

如果bean标签中包含用户自定义标签或者属性，那就需要在这里进行处理。
上一个方法 parseBeanDefinitionElement 用于解析默认标签的基本设置。

```java
// BeanDefinitionParserDelegate.class 1400
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder originalDef) {
    return decorateBeanDefinitionIfRequired(ele, originalDef, null);
}

// BeanDefinitionParserDelegate.class 1411
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
        Element ele, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {

    BeanDefinitionHolder finalDefinition = originalDef;

    //<bean id="12" xml:id="asdf"/>
    NamedNodeMap attributes = ele.getAttributes();
    for (int i = 0; i < attributes.getLength(); i++) {
        Node node = attributes.item(i);
        finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
    }

    //<bean id="12" xml:id="asdf">
    //    <dubbo:consumer check="false"/>
    //</bean>
    // 用于解析子标签中的用户自定义标签，上面的dubbo就是用户自定义标签。
    NodeList children = ele.getChildNodes();
    for (int i = 0; i < children.getLength(); i++) {
        Node node = children.item(i);
        if (node.getNodeType() == Node.ELEMENT_NODE) {
            finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
        }
    }
    return finalDefinition;
}
```

注册逻辑就简单了：
* 将beanName和beanDefinition注册到map中。
* 将beanName和所有alias也注册到map中。

```java
// BeanDefinitionReaderUtils.class 158
public static void registerBeanDefinition(
        BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
        throws BeanDefinitionStoreException {

    // 将beanName作为key，beanDefinition作为value注册到一个map中。
    String beanName = definitionHolder.getBeanName();
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // 将所有别名都注册一遍
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}
```

注册beanDefinition的逻辑需要稍微看一下。
需要判断beanName是否已经注册过，以及当前工厂是否已经开始初始化了。

```java
// DefaultListableBeanFactory.class 976
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException {

    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    // 执行校验方法
    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                    "Validation of bean definition failed", ex);
        }
    }

    // 判断这个beanName是否已经存在
    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {...}
    else {
        // 判断工厂是否已经开始初始化Bean了
        if (hasBeanCreationStarted()) {
            // 锁住beanDefinitionMap，将当前beanDefinition加到最后一个去。
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                removeManualSingletonName(beanName);
            }
        }
        else {
            // 工厂还没开始初始化，那就讲beanName和beanDefinition的关系添加到map中。
            // 将beanName添加到List中。
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            removeManualSingletonName(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }

    if (existingDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
    else if (isConfigurationFrozen()) {
        clearByTypeCache();
    }
}
```

**阶段性总结**

Spring是如何解析默认标签的呢？
默认标签有四种import、alias、bean、beans，我们主要关注bean标签的解析。
* 首先解析id和name属性，并生成对应的beanName
* 然后解析其余属性和子标签。
* 如果有用户自定义的属性和标签也需要解析一下，这是 decorateBeanDefinitionIfRequired 这个方法的作用。
* 将beanDefinition和别名alias注册到Spring的map中。

#### 解析用户自定义标签

这是Spring扩展性的优秀实现。

问：以Dubbo标签举例，每个包含Dubbo标签的XML文件头部都必须有哪两样东西？

> xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
> xsi:schemaLocation=http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd
> xmlns其实是xmlNameSpace，也就是xml的命名空间，你需要加了这个东西才能在文件中使用dubbo相关的xml标签。
> xsi:schemaLocation我理解是对你xml标签的限制。
> 例如dubbo的protocol标签里面的属性，在这个文件里面都确定好了，你不能写额外的属性。

问：上面是XML文件中的对dubbo标签的描述。那么即使我再XML文件中写了dubbo标签，这又该由谁来解析呢？毕竟Spring可是没有针对dubbo标签的解析器

> DubboNamespaceHandler 这个类里面就定义了如何解析Dubbo的各种标签。
> 它的init方法将标签名称和对应的解析器的关系注册到Spring。这样，在Spring解析xml文件的时候，发现标签的命名空间是用户自定义的，首先会通过命名空间URL获取到对应的命名空间解析器，然后再通过标签名获取到对应的标签解析器，然后执行解析操作。

问：还有一个问题，xml文件中的xmlns是如何与NameSpaceHandler关联起来的呢？

> 在Dubbo的jar包的META-INF有一个spring.handlers文件，里面将命名空间与命名空间解析器关联起来了。
> http\://dubbo.apache.org/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
> http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler

回到源码，假设现在正在解析这个标签

/<dubbo:registry address="www.baidu.com" timeout="10000" />

首先获取标签的URI，然后通过URI获取到对应的NameSpaceHandler，接着调用parse方法。

```java
// BeanDefinitionParserDelegate.class 1381
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
    // 获取标签的命名空间URI，也就是：http://code.alibabatech.com/schema/dubbo
    String namespaceUri = getNamespaceURI(ele);
    if (namespaceUri == null) {
        return null;
    }
    // 通过命名空间URI获取到对应的命名空间解析器，也就是：DubboNameSpaceHandler
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    if (handler == null) {
        error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
        return null;
    }
    // 从这里进去
    return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

Dubbo所有标签的解析用的都是一个方法，不同类型的标签入参的beanClass不同。

根据我的注释大致看一下就可以，基本就是把标签的所有参数做成键值对形式，放到beanDefinition的一个map中然后返回。

```java
// DubboBeanDefinitionParser.class 74
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
    RootBeanDefinition beanDefinition = new RootBeanDefinition();
    // beanClass在DubboNameSpcaeHandler定义的时候就设置好了，是这个：RegistryConfig.class
    beanDefinition.setBeanClass(beanClass);
    beanDefinition.setLazyInit(false);
    String id = element.getAttribute("id");
    // 标签的属性中没有设置id，那就自动生成一个。首先拿name中的属性，其次拿interface中的属性，最后用beanClass加一个数字。
    if ((id == null || id.length() == 0) && required) {...}
    if (id != null && id.length() > 0) {
        if (parserContext.getRegistry().containsBeanDefinition(id)) {
            throw new IllegalStateException("Duplicate spring bean id " + id);
        }
        // 注册beanDefinition
        parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
        beanDefinition.getPropertyValues().addPropertyValue("id", id);
    }
    if (ProtocolConfig.class.equals(beanClass)) {...}
    else if (ServiceBean.class.equals(beanClass)) {
        // 获取标签中的class属性
        String className = element.getAttribute("class");
        if (className != null && className.length() > 0) {
            RootBeanDefinition classDefinition = new RootBeanDefinition();
            // 通过类名获取类对象
            classDefinition.setBeanClass(ReflectUtils.forName(className));
            classDefinition.setLazyInit(false);
            // 解析这个标签包含的子标签。一般工作中很少用到这个
            parseProperties(element.getChildNodes(), classDefinition);
            beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
        }
    } 
    else if (ProviderConfig.class.equals(beanClass)) {...}
    else if (ConsumerConfig.class.equals(beanClass)) {...}
    Set<String> props = new HashSet<String>();
    ManagedMap parameters = null;
    // 没有细看
    for (Method setter : beanClass.getMethods()) {}
    // 获取标签的参数，我们大多的配置就是在这里被解析的
    NamedNodeMap attributes = element.getAttributes();
    int len = attributes.getLength();
    // 将属性名和对应的内容解析成键值对形式，放到parameters中。{"address":"com.baidu.com","timeout":"10000"}
    for (int i = 0; i < len; i++) {
        Node node = attributes.item(i);
        String name = node.getLocalName();
        if (!props.contains(name)) {
            if (parameters == null) {
                parameters = new ManagedMap();
            }
            String value = node.getNodeValue();
            parameters.put(name, new TypedStringValue(value, String.class));
        }
    }
    // 将上面解析到的所有参数放到paramemters里面。
    if (parameters != null) {
        beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
    }
    return beanDefinition;
}
```
