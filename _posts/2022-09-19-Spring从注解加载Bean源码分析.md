---
layout: default
title: "Spring从注解加载Bean源码分析"
data: 2022-09-19 16:11:00 -0000
category: spring
---

我最近工作中大多数配置都是用的注解方式定义。

想要你的注解被Spring扫描并管理，你需要在类层面加上一个@Component，想@Configuration内部也是有@Component的。

Spring的版本是 5.2.6.RELEASE

Dubbo版本是 2.6.2

本文是Spring IOC 初始化流程的一部分，回到初始化过程中的 refreshBeanFactory 方法。

本文分析从注解中加载的源码逻辑。

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

假设我们扫描的配置如下，它会注册Application.class和com.zimingsir包下所有被@Component注解的类。

```java
@ComponentScan(basePackageClasses = {Application.class}, basePackages = "com.zimingsir")
```


```java
// AnnotationConfigWebApplicationContext.java 198
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) {
    // 将指定类解析为BeanDefinition，并注册到beanFactory中
    AnnotatedBeanDefinitionReader reader = getAnnotatedBeanDefinitionReader(beanFactory);
    // 将指定路径下的类解析成BeanDefinition，并注册到beanFactory中
    ClassPathBeanDefinitionScanner scanner = getClassPathBeanDefinitionScanner(beanFactory);
    
    // beanName生成器
    BeanNameGenerator beanNameGenerator = getBeanNameGenerator();
    if (beanNameGenerator != null) {
        reader.setBeanNameGenerator(beanNameGenerator);
        scanner.setBeanNameGenerator(beanNameGenerator);
        beanFactory.registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
    }

    //用于解析Bean的@Scope注解，根据scope判断我们能否使用这个Bean
    ScopeMetadataResolver scopeMetadataResolver = getScopeMetadataResolver();
    if (scopeMetadataResolver != null) {
        reader.setScopeMetadataResolver(scopeMetadataResolver);
        scanner.setScopeMetadataResolver(scopeMetadataResolver);
    }
    
    // 将需要扫描的类注册到beanFactory中，也就是Application.class
    if (!this.componentClasses.isEmpty()) {
        if (logger.isDebugEnabled()) {
            logger.debug("Registering component classes: [" +
                    StringUtils.collectionToCommaDelimitedString(this.componentClasses) + "]");
        }
        reader.register(ClassUtils.toClassArray(this.componentClasses));
    }

    // 将指定路径下的BeanDefinition注册到beanFactory中，也就是com.baidu包下所有被@Component注解的类
    if (!this.basePackages.isEmpty()) {
        if (logger.isDebugEnabled()) {
            logger.debug("Scanning base packages: [" +
                    StringUtils.collectionToCommaDelimitedString(this.basePackages) + "]");
        }
        scanner.scan(StringUtils.toStringArray(this.basePackages));
    }

    // 获取配置
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        // 遍历所有配置
        for (String configLocation : configLocations) {
        
            // 首先尝试用class的方式去加载它
            try {
                Class<?> clazz = ClassUtils.forName(configLocation, getClassLoader());
                if (logger.isTraceEnabled()) {
                    logger.trace("Registering [" + configLocation + "]");
                }
                reader.register(clazz);
            }
            catch (ClassNotFoundException ex) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Could not load class for config location [" + configLocation +
                            "] - trying package scan. " + ex);
                }
                // class方式加载失败后，使用路径的方式去加载
                int count = scanner.scan(configLocation);
                if (count == 0 && logger.isDebugEnabled()) {
                    logger.debug("No component classes found for specified class/package [" + configLocation + "]");
                }
            }
        }
    }
}
```

上面代码的逻辑就是首先通过reader将指定的class类加载为BeanDefinition然后注册。

其次就是sacnner扫描路径下的所有包，将包中的类加载为BeanDefinition然后注册。

一个是通过class的方式，一个是通过路径的方式来扫描。

### 通过class的方式来注册

```java
// AnnotatedBeanDefinitionReader.class 135
public void register(Class<?>... componentClasses) {
    for (Class<?> componentClass : componentClasses) {
        registerBean(componentClass);
    }
}

public void registerBean(Class<?> beanClass) {
    doRegisterBean(beanClass, null, null, null, null);
}

// AnnotatedBeanDefinitionReader.class 249
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
                                @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
                                @Nullable BeanDefinitionCustomizer[] customizers) {

    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    abd.setInstanceSupplier(supplier);
    // 获取scope属性
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    // 生成beanName
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    // 解析一些基本注解，例如：@Lazy，@Primary，@DependsOn，@Role，@Description
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    if (customizers != null) {
        for (BeanDefinitionCustomizer customizer : customizers) {
            customizer.customize(abd);
        }
    }

    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 将beanName和beanDefinition的关系注册到Spring，并且将beanName和alias的关系注册到Spring。
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

### 通过路径的方式来注册Bean

从上一个方法的 scanner.scan() 追踪下去


```java
// ClassPathBeanDefinitionScanner.java 251
public int scan(String... basePackages) {
    // 纪录一下扫描之前的BeanDefinition数量
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

    // 扫描并注册指定路径下的所有BeanDefinition
    doScan(basePackages);

    // 暂时不清楚
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }
    
    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}

// ClassPathBeanDefinitionScanner.java 272
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
         // 获取路径下面的所有BeanDefinition
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
             // 设置Bean的自动依赖注入装配属性等
            if (candidate instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
             // 对一部分注解进行前置处理
            if (candidate instanceof AnnotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
            }
             // 判断beanName是否已经注册过
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                 // 不清楚这里的含义
                definitionHolder =
                        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                 // 注册beanDefinition，将beanName->beanDefinition注册到Map中。以及别名->beanName也注册到Map中
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}

```


接下来看一下它是如何将路径下的BeanDefinition解析出来的


```java
// ClassPathScanningCandidateComponentProvider.java 310
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
    }
    else {
        // 一般会走到这里
        return scanCandidateComponents(basePackage);
    }
}

// ClassPathScanningCandidateComponentProvider.java 415
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    try {
        // 转换路径，用于扫描路径下的所有class文件。将com.zimingsir 转换为 classpath*:com/zimingsir/**/*.class
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        // 将每个class文件解析成Resource
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
        boolean traceEnabled = logger.isTraceEnabled();
        boolean debugEnabled = logger.isDebugEnabled();
        for (Resource resource : resources) {
            if (traceEnabled) {
                logger.trace("Scanning " + resource);
            }
            if (resource.isReadable()) {
                try {
                    // 通过ASM获取class元数据，并封装在MetadataReader元数据读取器中
                    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                    //判断该类是否符合@CompoentScan的过滤规则
                    //过滤匹配排除excludeFilters排除过滤器(可以没有),包含includeFilter中的包含过滤器（至少包含一个）。
                    if (isCandidateComponent(metadataReader)) {
                        // 将数据转换为BeanDefinition
                        ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                        sbd.setSource(resource);
                        // 判断bean是否合格
                        if (isCandidateComponent(sbd)) {
                            if (debugEnabled) {
                                logger.debug("Identified candidate component class: " + resource);
                            }
                            // 加入到返回的集合中
                            candidates.add(sbd);
                        }
                        else {...}
                    }
                    else {...}
                }
                catch (Throwable ex) {...}
            }
            else {...}
        }
    }
    catch (IOException ex) {...}
    return candidates;
}
```

看一下处理通用注解都处理了一些什么东西。

**分析processCommonDefinitionAnnotations()的作用**
```java
// AnnotationConfigUtils.java 233
public static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd) {
    // 处理一些通用的注解
    processCommonDefinitionAnnotations(abd, abd.getMetadata());
}

// AnnotationConfigUtils.java 237
static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
    // 处理@Lazy注解。@Lazy注解起到延迟加载的作用，第一次使用的时候才会初始化Bean
    AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
    if (lazy != null) {
        abd.setLazyInit(lazy.getBoolean("value"));
    }
    else if (abd.getMetadata() != metadata) {
        lazy = attributesFor(abd.getMetadata(), Lazy.class);
        if (lazy != null) {
            abd.setLazyInit(lazy.getBoolean("value"));
        }
    }

    // 处理@Primary注解。当有多个相同类型的Bean的时候，我们可以指定其中一个Bean为主要的Bean，主要的Bean会被优先加载。
    if (metadata.isAnnotated(Primary.class.getName())) {
        abd.setPrimary(true);
    }
    
    // 处理@DependsOn注解。当前Bean依赖的另外一些bean，初始化当前bean的时候需要先初始化依赖的bean。
    AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
    if (dependsOn != null) {
        abd.setDependsOn(dependsOn.getStringArray("value"));
    }
    // 处理@Role注解。该注解用于标识Bean的分类，具体用法不清楚。
    AnnotationAttributes role = attributesFor(metadata, Role.class);
    if (role != null) {
        abd.setRole(role.getNumber("value").intValue());
    }
    // 处理@Description注解。该注解用于描述当前bean
    AnnotationAttributes description = attributesFor(metadata, Description.class);
    if (description != null) {
        abd.setDescription(description.getString("value"));
    }
}
```