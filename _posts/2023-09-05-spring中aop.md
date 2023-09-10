---
layout: post
title: spring中的AOP
---

## 什么是AOP？

面向切面编程,aop可以在程序执行的某一位置(join-point)执行切面逻辑,如在delete方法开始时打印log记录xx被删除了.

aop的实现方式是通过对目标对象进行代理(类似wrapper)
,对其他依赖目标对象的实例注入代理对象,当调用目标对象的方法时,会先对代理对象进行调用,在目标方法的执行前后执行对应的advice,完成切面处理.

### 相关概念:

- 切面(aspect): 横切面对象,一般为一个具体类对象(可以借助@Aspect声明)。
- 通知(Advice): 在切面的某个特定连接点上执行的动作(扩展功能)，例如around,before,after等。
- 连接点(joinpoint): 程序执行过程中某个特定的点，一般指向被拦截到的目标方法。
- 切入点(pointcut): 对多个连接点(Joinpoint)一种定义,一般可以理解为多个连接点的集合。
- 代理(proxy): 对目标对象实例进行包装,在目标对象方法执行前,执行advice逻辑.
- 动态代理: 运行时动态生成的代理对象,分为cglib(继承)和jdk动态代理(接口).
- 静态代理: 编译期生成的代理对应,如硬编码代码和aspectj,aspectj通过特定编译器对class文件进行增强.

### AspectJ Cglib Jdk动态代理的关系

cglib和jdk动态代理只用来生成代理对象.他们的功能算是Aspectj的子集.

Aspectj时一个全面的AOP包,包含定义切面(@Aspect等注解)和生成静态代理两部分功能,静态代理需要AJC(支持Aspect织入的编译器).

Spring AOP中默认只使用了前者,即注解部分的功能,借用Aspectj的注解解析当前声明了哪些切面,再使用cglib或jdk动态代理生成代理对象.

***spring boot2.0***之后默认实现方式改为`cglib`方式，之前为`jdk动态代理`(
参考 `org.springframework.boot.autoconfigure.aop.AopAutoConfiguration.AspectJAutoProxyingConfiguration.CglibAutoProxyConfiguration`)

## Spring AOP如何工作的？

### 1.SpringBoot中AOP的自动配置

#### `@EnableAspectJAutoProxy`
`@EnableAspectJAutoProxy`导入一个`AspectJAutoProxyRegistrar`
,参考 ![调用顺序](../images/spring-aop/callers-of-auto-config-register.png "调用顺序"),`AspectJAutoProxyRegistrar`会向bean factory注册`AnnotationAwareAspectJAutoProxyCreator`的对象,此对象的`#findCandidateAdvisors()`方法会扫描`@Aspectj`相关注解和`Advisor`, 其也是`InstantiationAwareBeanPostProcessor`的子类,
所以对象的代理创建会在`#AbstractAutoProxyCreator#postProcessAfterInitialization`和`AbstractAutoProxyCreator#postProcessBeforeInstantiation`中被调用(注意一个是实例化前,一个是初始化后)

我们看下`#postProcessAfterInitialization`方法的过程
```
	/**
	 * Create a proxy with the configured interceptors if the bean is
	 * identified as one to proxy by the subclass.
	 * @see #getAdvicesAndAdvisorsForBean
	 */
	@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
			        //对于一个bean尝试创建代理对象
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}


	/**
	 * Wrap the given bean if necessary, i.e. if it is eligible for being proxied.
	 * @param bean the raw bean instance
	 * @param beanName the name of the bean
	 * @param cacheKey the cache key for metadata access
	 * @return a proxy wrapping the bean, or the raw bean instance as-is
	 */
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	        //对于this.targetSourcedBeans.contains的bean,已经在实例化之前就代理过了
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		//已处理过,但无需代理
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		//基础bean,skip
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		//获取当前符合bean的advisor,创建代理
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}


	/**
	 * Create an AOP proxy for the given bean.
	 * @param beanClass the class of the bean
	 * @param beanName the name of the bean
	 * @param specificInterceptors the set of interceptors that is
	 * specific to this bean (may be empty, but not null)
	 * @param targetSource the TargetSource for the proxy,
	 * already pre-configured to access the bean
	 * @return the AOP proxy for the bean
	 * @see #buildAdvisors
	 */
	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}
                //用于创建代理的工厂,其中含有一些配置,如代理类型cglib还是jdk
		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
		        //此处就是代理类型判定了,true就是cglib方式
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
			        //jdk代理解析interface
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}
                //至此代理创建完成
		return proxyFactory.getProxy(getProxyClassLoader());
	}
```


此对象的`#findCandidateAdvisors()`方法会扫描上下文中的存在`@Aspecj`注解和`Advisor`实例.

```java
        //AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors
        @Override
	protected List<Advisor> findCandidateAdvisors() {
		// Add all the Spring advisors found according to superclass rules.
                // 父类 AbstractAdvisorAutoProxyCreator 的方法,其会扫描所有的 Advisor子类的实例
		List<Advisor> advisors = super.findCandidateAdvisors();
		// Build Advisors for all AspectJ aspects in the bean factory.
		if (this.aspectJAdvisorsBuilder != null) {
                        // BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors() 用于扫描所有的Aspectj注解
			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
		}
		return advisors;
	}
```
