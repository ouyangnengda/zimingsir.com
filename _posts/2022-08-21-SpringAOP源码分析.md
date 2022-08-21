---
layout: default
title: "SpringAOP源码分析"
data: 2022-08-21 20:40:00 -0000
category: spring
---

### 面向切面编程
咱们大家都知道透析吧，血液从身体里面流出来，经过一个罐子，罐子里面摆着一层层的透析薄膜，血液流过透析薄膜，废物被透析薄膜拦截，干净的血流流回我们的身体。我们JAVA中的AOP思想就像这一个透析罐子，AOP中的切面就像是罐子里面一层层的透析薄膜，流经里面的血液就像是切点。

就我暂时所遇到的编程来说：它们都有一个共同点，那就是它们一直在尽量减少重复的代码。线程池的责任是这样，NIO中的多路复用器是这样，数据库连接池是这样，AOP是这样，IOC还是这样。

**AOP， Spring AOP，AspectJ这三者关系**

AOP是一种思想，Spring AOP 和 AspectJ 是对这种思想的一个具体实现，Spring AOP 是基于 Spring Bean 做的实现，而 AspectJ 则是可以针对Java字节码进行修改。

@Pointcut() 注解的匹配方式：
通常 "." 代表一个包名，".." 代表包及其子包，方法参数任意匹配使用两个点 ".."。


### 梗概

AOP：面向切面编程
解决了什么问题：在不改变原有逻辑的情况下，增强代码，从根本上解决代码重复问题。

面向切面编程支持两种方式：JDK动态代理和CGLIB动态代理。

**JDK动态代理面向接口**
1. 声明一个类继承InvocationHandler这个接口，实现invoker方法。
2. 在invoke方法中实现我们的代理逻辑
3. 通过Proxy.newProxyInstance传入代理接口的ClassLoader、Hello.class、InvocationHandler。生成一个代理类，强转为接口。

**CGlib动态代理面向子类**
cglib可以在运行期扩展Java类的实现。
cglib封装了asm，可以在运行期动态生成新的class（子类）

五种增强：
1. 前置增强：在目标方法执行前
2. 后置增强：在目标方法执行后
3. 环绕增强：在目标方法执行前后都进行
4. 异常抛出增强：抛出异常后执行
5. 引介增强：为目标类增加新的方法和属性

**如何实现一个切面**
1. 在配置类配置@EnableAspectJAutoProxy
2. 使用@Aspect注解那个AOP类
3. @Pointcut("execution(* Concert.Performance.perform(..))") 修饰一个方法用于确定切面
4. @Around("performance()") 确定环绕方式和织入对象

### 深入源码

DefaultAdvisorAutoProxyCreator 将这个类注入为Bean可以将所有的 Advicor 生效。

Spring 5.3.6

DefaultAdvisorAutoProxyCreator 这个类继承了BeanPostProcessor，实现了 postProcessAfterInitialization() 方法，在类完成初始化之后会对类进行动态代理，然后返回代理之后的对象。具体见：AbstractAutowireCapableBeanFactory.class 437行

```java
// AbstractAutoProxyCreator.class 286
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
		// 获取bean的唯一标识
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

首先通过bean的Class对象或者beanName获取它的唯一标识

接着调用 wrapIfNecessary 方法，如果需要增加就先增强然后返回代理类。

```java
// AbstractAutoProxyCreator.class 325
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
	// 这个bean不需要增强就直接返回
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
	// 如果bean是一个增强、切点等等则不会被代理。第二种情况是IOC的一个特性，以.ORIGINAL作为class路径结尾的也不会被代理。
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // 获取这个类的增强
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
		// 将bean的代理标记设置为true
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
		// 将增强加入到代理类中
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

wrapIfNecessary 就是Spring AOP的核心方法了

首先判断这个bean是否需要增强，不需要增强的话直接返回原始的bean。

接着判断这个bean是不是本身就是一个AOP相关的类：Advice，Pointcut，Advisor，AopInfrastructureBean 这种情况也不需要增强。

或者这个bean的class以.ORIGINAL结尾，这种情况也不会增强。这是IOC的一种特性，具体不清楚

然后获取这个类的增强列表

将这个bean在advisedBeans设置为已代理

将增强列表加入到代理类中

返回代理类。

从上面的逻辑看，核心的方法就是获取bean对应的增强，然后将增强加入到代理类中。下文将按照这两个部分分别分析

### 获取bean对应的增强

getAdvicesAndAdvisorsForBean 获取bean对应的增强

它的实现类是 AbstractAdvisorAutoProxyCreator

```java
// AbstractAdvisorAutoProxyCreator.class 75
protected Object[] getAdvicesAndAdvisorsForBean(
        Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
	// 获取这个bean的增强列表
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}

// AbstractAdvisorAutoProxyCreator.class 95
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    // 获取候选的增强列表
	List<Advisor> candidateAdvisors = findCandidateAdvisors();
	// 从候选增强列表挑选出当前bean可以使用的
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
	// 钩子方法，留给子类使用。
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```

上面方法的逻辑先拿到候选的增强列表

然后从候选的增强列表选出当前bean可以使用的返回。

钩子方法，留给子类使用

将增强排序后返回

那么它是如何获取到候选的增强列表的呢？

```java
// AbstractAdvisorAutoProxyCreator.class 109
protected List<Advisor> findCandidateAdvisors() {
	Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
	// 获取增强列表。增强也是以bean的方式被Spring管理
	return this.advisorRetrievalHelper.findAdvisorBeans();
}

// BeanFactoryAdvisorRetrievalHelper.class 67
public List<Advisor> findAdvisorBeans() {
	String[] advisorNames = this.cachedAdvisorBeanNames;
	// 如果增强的beanName列表是空的，那就进行初始化
	if (advisorNames == null) {
		// 获取当前beanFactory中的属于增强类的beanName。并且允许获取singleton类型之外的bean。
		advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
				this.beanFactory, Advisor.class, true, false);
		this.cachedAdvisorBeanNames = advisorNames;
	}
	if (advisorNames.length == 0) {
		return new ArrayList<>();
	}

	List<Advisor> advisors = new ArrayList<>();
	// 遍历增强beanName列表
	for (String name : advisorNames) {
		if (isEligibleBean(name)) {
			// beanName正在创建中则跳过
			if (this.beanFactory.isCurrentlyInCreation(name)) {
				if (logger.isTraceEnabled()) {
					logger.trace("Skipping currently created advisor '" + name + "'");
				}
			}
			else {
				try {
					// 初始化beanName
					advisors.add(this.beanFactory.getBean(name, Advisor.class));
				}
				catch (BeanCreationException ex) {
					Throwable rootCause = ex.getMostSpecificCause();
					if (rootCause instanceof BeanCurrentlyInCreationException) {
						BeanCreationException bce = (BeanCreationException) rootCause;
						String bceBeanName = bce.getBeanName();
						if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
							if (logger.isTraceEnabled()) {
								logger.trace("Skipping advisor '" + name +
										"' with dependency on currently created bean: " + ex.getMessage());
							}
							// Ignore: indicates a reference back to the bean we're trying to advise.
							// We want to find advisors other than the currently created bean itself.
							continue;
						}
					}
					throw ex;
				}
			}
		}
	}
	return advisors;
}
```

上面的逻辑就是先获取当前beanFactory所有的增强类的beanName，然后逐个调用getBean方法初始化。

上面方法中获取增强类beanName的逻辑就不深入了。

回到主线逻辑，我们拿到了所有的候选增强之后，接下来就是从中选中出当前beanName可以使用的增强，然后进行增强操作。

那么它是如何从候选的增强列表挑选出当前bean可以使用的呢？

```java
// AbstractAdvisorAutoProxyCreator.class 95
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    // 获取候选的增强列表
	List<Advisor> candidateAdvisors = findCandidateAdvisors();
	// 从候选增强列表挑选出当前bean可以使用的
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
	// 钩子方法，留给子类使用。
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```

```java
// AbstractAdvisorAutoProxyCreator.class 109
protected List<Advisor> findAdvisorsThatCanApply(
		List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {
	// 将beanName标记为正在进行代理
	ProxyCreationContext.setCurrentProxiedBeanName(beanName);
	try {
		// 返回当前bean可用的增强
		return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
	}
	finally {
		// 取消标记
		ProxyCreationContext.setCurrentProxiedBeanName(null);
	}
}

// AopUtils.class 305
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
	if (candidateAdvisors.isEmpty()) {
		return candidateAdvisors;
	}
	List<Advisor> eligibleAdvisors = new ArrayList<>();
	for (Advisor candidate : candidateAdvisors) {
		// IntroductionAdvisor 不太清楚，我们常用的是PointcutAdvisor，在下面应用
		if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
			eligibleAdvisors.add(candidate);
		}
	}
	boolean hasIntroductions = !eligibleAdvisors.isEmpty();
    // 遍历所有增强，如果某个增强可以应用clazz的任何一个方法则将该增强加入到eligibleAdvisors中。
	for (Advisor candidate : candidateAdvisors) {
		if (candidate instanceof IntroductionAdvisor) {
			// already processed
			continue;
		}
		// PointcutAdvisor 在这里被应用
		if (canApply(candidate, clazz, hasIntroductions)) {
			eligibleAdvisors.add(candidate);
		}
	}
	return eligibleAdvisors;
}
```

**IntroductionAdvisor和PointcutAdvisor有什么不同？**
PointcutAdvisor切点的粒度是方法，它是在某一个方法执行的前后加上我们的增强。
而IntroductionAdvisor切点的粒度是类


```java
// AopUtils.class 283
public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
	if (advisor instanceof IntroductionAdvisor) {
		return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
	}
	// 常用的是这个
	else if (advisor instanceof PointcutAdvisor) {
		PointcutAdvisor pca = (PointcutAdvisor) advisor;
		return canApply(pca.getPointcut(), targetClass, hasIntroductions);
	}
	else {
		// It doesn't have a pointcut so we assume it applies.
		return true;
	}
}

// AopUtils.class 224
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
	Assert.notNull(pc, "Pointcut must not be null");
	// 
	if (!pc.getClassFilter().matches(targetClass)) {
		return false;
	}
	
	// 获取方法匹配器，看看这个切点可以匹配上这个类的某一个方法吗？可以匹配上任何一个就需要对这个类进行代理。
	MethodMatcher methodMatcher = pc.getMethodMatcher();
	if (methodMatcher == MethodMatcher.TRUE) {
		// No need to iterate the methods if we're matching any method anyway...
		return true;
	}

	IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
	if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
		introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
	}

	Set<Class<?>> classes = new LinkedHashSet<>();
	if (!Proxy.isProxyClass(targetClass)) {
		classes.add(ClassUtils.getUserClass(targetClass));
	}
	// 获取这个类实现的所有接口
	classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

	// 如果这个切点可以匹配这个类中的任何一个方法则返回true，否则返回false。
	for (Class<?> clazz : classes) {
		Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
		for (Method method : methods) {
			if (introductionAwareMethodMatcher != null ?
					introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
					methodMatcher.matches(method, targetClass)) {
				return true;
			}
		}
	}

	return false;
}
```

从所有候选增强中返回当前bean可用的增强列表，这个逻辑其实也比较直接。

遍历所有候选增强，遍历的时候判断这个增强是否可以匹配bean对应类的任何一个方法，如果可以则返回true将该增强加入到返回列表中。

**总结**
1. 从当前beanFactory中获取所有Advisor.class类型的bean，这些就是候选增强
2. 将上述的所有bean逐个调用getBean方法，进行初始化。
3. 遍历所有增强
4. 遍历的时候判断当前增强是否可以匹配目标class的任何一个方法，如果可以则返回true，表示当前增强可用于该class，接着将当前增强加入到可用增强列表。

下面需要将可用增强列表增强到目标类中。

### 生成代理

回到最开始的方法，getAdvicesAndAdvisorsForBean 这个方法已经分析完了，接下来看下createProxy方法。

```java
// AbstractAutoProxyCreator.class 325
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
	// 这个bean不需要增强就直接返回
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
	// 如果bean是一个增强、切点等等则不会被代理。第二种情况是IOC的一个特性，以.ORIGINAL作为class路径结尾的也不会被代理。
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // 获取这个类的增强
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
		// 将bean的代理标记设置为true
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
		// 将增强加入到代理类中
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

```java
// AbstractAutoProxyCreator.class 433
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
							 @Nullable Object[] specificInterceptors, TargetSource targetSource) {

	if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
		AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
	}

	ProxyFactory proxyFactory = new ProxyFactory();
	proxyFactory.copyFrom(this);

	if (!proxyFactory.isProxyTargetClass()) {
		// proxy-target-class="true" 如果有这个配置表示要求用CGLIB生成代理
		if (shouldProxyTargetClass(beanClass, beanName)) {
			proxyFactory.setProxyTargetClass(true);
		}
		else {
			// beanClass有接口的话把接口都添加到proxyFactory中；没有接口就setProxyTargetClass(true)，表示使用CGLIB
			evaluateProxyInterfaces(beanClass, proxyFactory);
		}
	}
	// 将入参的增强和公共的增强包装成Advisor数组，并返回
	Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
	proxyFactory.addAdvisors(advisors);
	proxyFactory.setTargetSource(targetSource);
	customizeProxyFactory(proxyFactory);

	proxyFactory.setFrozen(this.freezeProxy);
	if (advisorsPreFiltered()) {
		proxyFactory.setPreFiltered(true);
	}

	// Use original ClassLoader if bean class not locally loaded in overriding class loader
	ClassLoader classLoader = getProxyClassLoader();
	if (classLoader instanceof SmartClassLoader && classLoader != beanClass.getClassLoader()) {
		classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();
	}
	// 代理
	return proxyFactory.getProxy(classLoader);
}
```

生成代理工厂，根据配置使用CGLIB还是JDK动态代理，将入参的增强和公共的增强包装在一起返回一个增强数组，进行代理。


首先会通过代理工厂创建一个CGLIB或者JDK代理，然后调用他们实现的getProxy方法

```java
// ProxyFactory.class 109
public Object getProxy(@Nullable ClassLoader classLoader) {
	return createAopProxy().getProxy(classLoader);
}

// ProxyCreatorSupport.class 101
protected final synchronized AopProxy createAopProxy() {
	if (!this.active) {
		activate();
	}
    // 创建AOP代理类
	return getAopProxyFactory().createAopProxy(this);
}

// DefaultAopProxyFactory.class 53
@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    // 第一个参数不清楚，后面的参数是这些意思：是否要对代理进行优化 || proxy-target-class=true(使用CGLIB) || 代理对象没接口
	if (!NativeDetector.inNativeImage() &&
			(config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))) {
		Class<?> targetClass = config.getTargetClass();
		if (targetClass == null) {
			throw new AopConfigException("TargetSource cannot determine target class: " +
					"Either an interface or a target is required for proxy creation.");
		}
        // 代理对象就是接口
		if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
			return new JdkDynamicAopProxy(config);
		}
		return new ObjenesisCglibAopProxy(config);
	}
	else {
		return new JdkDynamicAopProxy(config);
	}
}
```


接下来就分别看CGLIB的getProxy方法和JDK动态代理的getProxy方法的实现了。


#### JDK动态代理

```java
// JdkDynamicAopProxy.class 122
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
	if (logger.isTraceEnabled()) {
		logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
	}
	// 通过构造对象生成一个代理对象，入参this就是InvocationHandler，它具体实现了invoke方法
	return Proxy.newProxyInstance(classLoader, this.proxiedInterfaces, this);
}
```

通过构造对象生成代理对象，实际的操作得看 JdkDynamicAopProxy 的 invoke 方法

```java
// JdkDynamicAopProxy.class 159
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Object target = null;

	try {
		// 如果当前调用的方法是equals或者hashCode且这两个方法没有被代理，则直接调用。 另外两个分支不清楚
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {...}
		else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {...}
		else if (method.getDeclaringClass() == DecoratingProxy.class) {....}
		else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {...}

		Object retVal;
		
		// 通过线程本地变量解决代理类内部方法自调用问题。@EnableAspectJAutoProxy(exposeProxy = true)
		// 详情见文章最后的学习文章列表
		if (this.advised.exposeProxy) {
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}


		target = targetSource.getTarget();
		Class<?> targetClass = (target != null ? target.getClass() : null);

		// 获取这个方法的增强列表。
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		// 如果增强列表为空，说明不需要增强，那我们不创建MethodInvocation，直接调用目标方法。
		if (chain.isEmpty()) {
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
		}
		else {
			// 创建一个MethodInvocation，它的构造方法只做了一些属性的赋值
			MethodInvocation invocation =
					new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			// 调用目标方法，肯定是先执行前置的方法，然后执行目标方法，接着执行后置方法。
			retVal = invocation.proceed();
		}

		// Massage return value if necessary.
		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target &&
				returnType != Object.class && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// Special case: it returned "this" and the return type of the method
			// is type-compatible. Note that we can't help if the target sets
			// a reference to itself in another returned object.
			retVal = proxy;
		}
		else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
			throw new AopInvocationException(
					"Null return value from advice does not match primitive return type for: " + method);
		}
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			// Must have come from TargetSource.
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```

看一下它是怎么获取这个方法对于的增强的，解析一下 getInterceptorsAndDynamicInterceptionAdvice

```java
// AdvisedSupport.class 466
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
	AdvisedSupport.MethodCacheKey cacheKey = new AdvisedSupport.MethodCacheKey(method);
	// 从缓存中获取当前方法对应的增强
	List<Object> cached = this.methodCache.get(cacheKey);
	//	缓存中没有拿到，则先生成然后放缓存里
	if (cached == null) {
		cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
				this, method, targetClass);
		this.methodCache.put(cacheKey, cached);
	}
	return cached;
}

// DefaultAdvisorChainFactory.class 51
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
		 Advised config, Method method, @Nullable Class<?> targetClass) {

	 AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
	 // 获取当前类匹配的增强，我们要从其中挑选出当前方法可以用的增强。
	 Advisor[] advisors = config.getAdvisors();
	 List<Object> interceptorList = new ArrayList<>(advisors.length);
	 Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
	 Boolean hasIntroductions = null;
	
	// 遍历所有增强，校验是否可以应用当前方法
	 for (Advisor advisor : advisors) {
	 	// 最常用的就是Pointcut增强，它是针对方法的增强
		 if (advisor instanceof PointcutAdvisor) {
			 // Add it conditionally.
			 PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
			 // 是否经历过预处理（默认false），如果没经历过那么现在进行匹配操作。判断增强是否可以作用于当前类
			 if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
				 MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
				 boolean match;
				 if (mm instanceof IntroductionAwareMethodMatcher) {
					 if (hasIntroductions == null) {
						 hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
					 }
					 match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
				 }
				 else {
				 	// 判断增强是否可以作用于当前方法
					 match = mm.matches(method, actualClass);
				 }
				 // 匹配成功
				 if (match) {
				 	// 将增强转化为MethodInterceptor方法拦截器
					 MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					 if (mm.isRuntime()) {
						 for (MethodInterceptor interceptor : interceptors) {
							 interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
						 }
					 }
					 else {
					 	// 添加到返回参数中
						 interceptorList.addAll(Arrays.asList(interceptors));
					 }
				 }
			 }
		 }
		 else if (advisor instanceof IntroductionAdvisor) {
			 IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
			 // IntroductionAdvisor的切点是类，只要类匹配成功就可以了
			 if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
				 Interceptor[] interceptors = registry.getInterceptors(advisor);
				 interceptorList.addAll(Arrays.asList(interceptors));
			 }
		 }
		 else {
			 Interceptor[] interceptors = registry.getInterceptors(advisor);
			 interceptorList.addAll(Arrays.asList(interceptors));
		 }
	 }

	 return interceptorList;
 }
```


接下来就是用递归的方式逐个调用列表中的拦截器，最后调用被代理的方法，然后返回递归方法，在返回递归的过程中实现后置增强。

```java
// ReflectiveMethodInvocation.class 160
public Object proceed() throws Throwable {
	// currentInterceptorIndex从-1开始，到size()-2结束，因此到size()-1就可以调用被代理的方法了。
	if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
		return invokeJoinpoint();
	}
	// 获取拦截器
	Object interceptorOrInterceptionAdvice =
			this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
	if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {

		InterceptorAndDynamicMethodMatcher dm =
				(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
		Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
		if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
			return dm.interceptor.invoke(this);
		}
		else {
			// Dynamic matching failed.
			// Skip this interceptor and invoke the next in the chain.
			return proceed();
		}
	}
	else {
		// 使用递归的方式去调用拦截器
		return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	}
}
```

看下 MethodInterceptor 的前置增强和后置增强的实现，还是挺简单的。

```java
// 前置增强
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {

	private final MethodBeforeAdvice advice;

	public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}

	@Override
	@Nullable
	public Object invoke(MethodInvocation mi) throws Throwable {
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
		return mi.proceed();
	}

}

// 后置增强
public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice, Serializable {

	private final AfterReturningAdvice advice;

	public AfterReturningAdviceInterceptor(AfterReturningAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}

	@Override
	@Nullable
	public Object invoke(MethodInvocation mi) throws Throwable {
		Object retVal = mi.proceed();
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		return retVal;
	}

}
```


**学习文章**

https://www.javadoop.com/post/spring-aop-source 了解了 Spring AOP 的入口方法

https://blog.csdn.net/f641385712/article/details/89303088 IntroductionAvisor相关知识

https://blog.csdn.net/u013720069/article/details/112025020 了解了AOP动态代理exposeProxy参数的意义

https://cloud.tencent.com/developer/article/1497782 学习了PointcutAdvisor和IntroductionAdvisor的区别