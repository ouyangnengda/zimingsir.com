---
layout: default
title: "SpringIOC源码分析"
data: 2022-07-29 22:58:00 -0000
category: spring
---

Spring的版本是 5.2.6.RELEASE

spring-boot-starter-parent的版本是 2.3.0.RELEASE

要学习IOC流程，自然是从 AbstractApplicationContext类的refresh方法开始，这是IOC容器初始化的骨架方法。

### refresh()

```java
// AbstractApplicationContext.java 516
public void refresh() throws BeansException, IllegalStateException {
    //加个锁，确保在你构建这个容器的时候别人不能销毁或者重新构建这个容器
    synchronized(this.startupShutdownMonitor) {

        //准备工作：记录现在时间、设置容器状态为active、设置打印日志的方式、
        this.prepareRefresh();

        // 这个方法做两个事，生成BeanFactory还有将用户定义的Bean解析为BeanDefinition
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();

        //准备工作：设置类加载器、添加BeanPostProcessor、注册系统bean、
        //确保存在Environment Bean，确保存在Properties Bean，确保存在SystemEnvironment Bean
        this.prepareBeanFactory(beanFactory);

        try {
            // 钩子方法
            this.postProcessBeanFactory(beanFactory);

            //调用BeanFactory所有子类的postProcessBeanFactory方法，这时候仅仅完成了注册还没初始化
            this.invokeBeanFactoryPostProcessors(beanFactory);

            //注册BeanPostProcessor
            this.registerBeanPostProcessors(beanFactory);

            // 以后来分析
            this.initMessageSource();

            //创建Spring的事件广播器。
            // 详细解析见：https://zimingsir.com/spring/Spring事件广播源码分析.html
            this.initApplicationEventMulticaster();

            // 模板方法
            this.onRefresh();

            //给事件广播器注册监听者，并且广播之前存储下来的事件。
            //详细解析见：https://zimingsir.com/spring/Spring事件广播源码分析.html
            this.registerListeners();

            //初始化所有的bean
            this.finishBeanFactoryInitialization(beanFactory);
            this.finishRefresh();
        } catch (BeansException var9) {...} 
        finally {
            this.resetCommonCaches();
        }

    }
}

```

总结一下大致做了这样几件事。

* 加锁。为了避免开始初始化之后被重新初始化
* 准备工作，标记当前状态，纪录当前时间。
* 获取BeanFactory
* 将各种形式定义的Bean解析为BeanDefinition
* BeanFactory的postProcess相关的注册和回调
* Spring事件通知相关
* 最后是初始化所有非延迟加载的单例Bean

### obtainFreshBeanFactory

从IOC的refresh方法进入到obtainFreshBeanFactory()

```java

// AbstractRefreshableApplicationContext 123
@Override
protected final void refreshBeanFactory() throws BeansException {
    
    //如果这个ApplicationContext已经存在
   if (hasBeanFactory()) {

      //销毁所有的单例Bean
      destroyBeans();

      //将BeanFactory变量指向一个null，序列化id设置为null
      closeBeanFactory();
   }
   try {

      //相当于new，不过多设置一个父BeanFactory，以后要一直使用这个DefaultListableBeanFactory
      DefaultListableBeanFactory beanFactory = createBeanFactory();
      beanFactory.setSerializationId(getId());

      //允许Bean重写，允许Bean循环依赖
      customizeBeanFactory(beanFactory);

      //解析各种形式定义的bean为BeanDefinition，并注册到beanFactory中。
      //后续会分析解析XML和解析注解的源码
      loadBeanDefinitions(beanFactory);
      synchronized (this.beanFactoryMonitor) {
         this.beanFactory = beanFactory;
      }
   }
   catch (IOException ex) {...}
}

```

loadBeanDefinitions()加载BeanDefinition大致有两种实现。

通过注解加载BeanDefinition：// 另一篇文章

通过XML加载BeanDefinition：// 另一篇文章

为了让当前文章的思路清晰，因此将上述两个加载流程的分析分别放到两篇文章。

当前我们只需要明白loadBeanDefinitions()方法就是将我们配置的Bean加载成BeanDefinition就可以。


### Bean初始化

```java
// AbstractApplicationContext.java 850
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 初始化conversion bean。这个东西是用来做转换的，不太熟悉
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }
    beanFactory.setTempClassLoader(null);
    beanFactory.freezeConfiguration();

    // 加载所有非延迟加载的单例Bean
    beanFactory.preInstantiateSingletons();
}

// DefaultListableBeanFactory.java 911
@Override
public void preInstantiateSingletons() throws BeansException {
    
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // 去初始化所有的非延迟加载的单例Bean，当然也是非抽象的
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {...}
            else {
                getBean(beanName);
            }
        }
    }

    // 如果bean继承自SmartInitializingSingleton，还得回调一下afterSingletonsInstantiated()这个方法
    for (String beanName : beanNames) {...}
}
```

上面就是遍历所有beanName去初始化

### 生成Bean

```java
// AbstractBeanFactory.java 201
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}

// AbstractBeanFactory.java 248
protected <T> T doGetBean(String name, ...) throws BeansException {

    // 如果传的是别名或者FactoryBean需要先转换成对应的beanName
    String beanName = transformedBeanName(name);
    Object beanInstance;

    // 从缓存中获取单例Bean
    Object sharedInstance = getSingleton(beanName);

    // 如果从缓存中拿到了直接返回
    if (sharedInstance != null && args == null) {

        // 如果是FactoryBean则返回它生产的那个Bean
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // 如果Bean是Property类型且正在创建中，说明发生了循环依赖。
        // 这种情况的循环依赖Spring无法解决，因此抛出异常。
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
        BeanFactory parentBeanFactory = getParentBeanFactory();

        // 父BeanFactory存在 && beanName不在当前的BeanFactory中，那么就去父BeanFactory中获取
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {...}

        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {

            // 获取合并后的BeanDefinition
            RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // 获取当前BeanDefinition依赖的BeanDefinition
            String[] dependsOn = mbd.getDependsOn();

            // 如果依赖了别的BeanDefinition，那么先将依赖的BeanDefinition全部创建好
            if (dependsOn != null) {...}

            // 依赖的BeanDefinition创建好了之后，开始创建当前的

            // 如果当前是单例
            if (mbd.isSingleton()) {

                // 这是首先执行 getSingleton 方法
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                // 普通的Bean什么也不做直接返回，如果是BeanFactory，返回它生产的类。
                beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

            else if (mbd.isPrototype()) {...}

            else {...}
        }
        catch (BeansException ex) {...}
        finally {
            beanCreation.end();
        }
    }

    // 检查Bean的类型是否对应
    return adaptBeanInstance(name, beanInstance, requiredType);
}

```

上面这一段逻辑就是：

首先去转换beanName的名称。

接着去缓存里面拿，如果拿到了那就返回。

如果没有拿到那就生成一个然后返回。

```java
// DefaultSingletonBeanRegistry.java 214
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    // 锁住缓存
    synchronized (this.singletonObjects) {
        // 尝试从缓存中获取beanName
        Object singletonObject = this.singletonObjects.get(beanName);

        // 如果没有从缓存中拿到那就创建
        if (singletonObject == null) {

            // 将beanName加入到正在创建的beanName的集合中
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;

            try {
                // 调用 createBean 去创建Bean
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            catch (IllegalStateException ex) {...}
            finally {
                // 将beanName从正在创建的beanName的集合中移除
                afterSingletonCreation(beanName);
            }

            // 如果是从新创建的，那就加入到缓存中
            if (newSingleton) {
                addSingleton(beanName, singletonObject);
            }
        }

        // 如果从缓存中拿到了那就用缓存的，否则就从新创建一个然后返回
        return singletonObject;
    }
}

```

singletonObjects 是一级缓存，它存储的已经完成实例化、填充数据、初始化的单例Bean。

如果一个缓存这没有拿到对象，那就需要创建一个了。

将当前bean加入到正在创建中。

创建当前bean。

移除标记。

加入到一级缓存中。

```java

// AbstractAutowireCapableBeanFactory.java 485
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    RootBeanDefinition mbdToUse = mbd;
    
    try {
        // 让 InstantiationAwareBeanPostProcessor 在这里有机会返回代理对象，如果返回了对象，方法就直接返回了。
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {...}

    try {
        // 创建Bean
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);

        return beanInstance;
    }
    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {...}
    catch (Throwable ex) {...}
}
```

给InstantiationAwareBeanPostProcessor一个返回对象的机会，如果不需要，那就我们创建Bean。

```java

// AbstractAutowireCapableBeanFactory.java 555
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 实例化Bean
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }

    // 提前暴露：实例化之后就将Bean放到第三级的缓存 singletonFactories 中。
    // 后面需要get这个Bean的时候就可以从第三级缓存中拿到了。
    // 当前还没有发生循环依赖，这只是第一次创建，这是为了解决后续可能的循环依赖而做的操作。
    // 第三个判断的 beanName 在前面的 DefaultSingletonBeanRegistry.java 227行加入到了列表中
    //                             当前Bean为单例Bean &&      允许循环依赖             &&     当前单例Bean正在创建中（发生了循环依赖）
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // 将Bean加入到三级缓存
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    Object exposedObject = bean;
    try {
        // 填充 Bean 内部的属性。循环依赖就是 A - B 之间的相互引用，那就是从这个方法开始的。
        populateBean(beanName, mbd, instanceWrapper);

        // 初始化
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {...}

    if (earlySingletonExposure) {...}

    return exposedObject;
}

```

首先实例化Bean，接着填充Bean中的属性，然后是初始化Bean。


```java

// 初始化
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 调用所有 BeanPostProcessor 的前置方法。所有的 BeanPostProcessor 都在 refresh 方法里面被注册了.
        // AbstractApplicationContext.java 567
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 前面是实例化和填充属性，现在是初始化。
        // 调用 InitializingBean 的 afterProperties 方法和 init-method xml 标签的方法。
        // 题外话：Dubbo的 ServiceBean 的暴露用的就是 afterProperties 方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 调用BeanPostProcessor的后置方法
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}

```

初始化就是BeanPostProcessor的前后钩子方法中间包着一个初始化。

初始化先调用 InitializingBean的afterPropertiesSet方法，接着调用init-method XMl 标签的方法。

### 学习IOC过程中收获很多的文章

https://www.javadoop.com/post/spring-ioc 对IOC的整体流程有一个较好的认识

https://blog.csdn.net/cristianoxm/article/details/107311570 通过这个文章学会了Spring是如何解析注解的。