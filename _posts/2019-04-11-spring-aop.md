---
layout:     post
title:      Spring源码阅读（二）
subtitle:   AOP
date:       2019-04-11
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Spring
---

Spring AOP的通常都是由BeanPostProcessor的实现类对bean进行自定义增强，基于AspectJ注解的BeanPostProcessor实现类是AnnotationAwareAspectJAutoProxyCreator，该类可以根据注解自动创建代理，可以查看该类的父类覆写的自定义增强方法：

```java
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}

	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
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

```
Spring AOP的核心在于如何将横切逻辑织入到关注点中，织入的关键在AopProxy接口，该接口定义如下：

```java
public interface AopProxy {

	Object getProxy();

	Object getProxy(@Nullable ClassLoader classLoader);
}
```

Spring AOP使用AopProxy对不同的代理实现进行了适度的抽象，针对不同的代理实现提供相应的AopProxy子类实现。通常的实现类有两种：JdkDynamicAopProxy以及CglibAopProxy(Spring4.0之后使用 ObjenesisCglibAopProxy），这两者分别使用JDK的动态代理和CGLIB动态代理生成织入对象的子类。不同AopProxy实现的实例化过程采用抽象工厂模式进行封装， 即通过AopProxyFactory进行。我们来看AopProxyFactory接口的定义：

```java
public interface AopProxyFactory {

	AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException;
}
```

AopProxyFactory通过传入的AdvisedSupport实例提供的相关信息，来决定生成什么类型的AopProxy，我们可以看它的具体实现类DefaultAopProxyFactory：

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        //如果需要优化或者没有实现接口，大部分情况下使用过Cglib动态代理
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
            //目标对象是接口或者是代理类，则仍使用Jdk动态代理
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
        //否则使用Jdk动态代理
		else {
			return new JdkDynamicAopProxy(config);
		}
	}

	//判断织入对象是否实现了接口
	private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
		Class<?>[] ifcs = config.getProxiedInterfaces();
		return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
	}
}
```

可见AopProxyFactory在创建织入对象的代理类时，如果织入对象没有实现接口，一般是使用Cglib动态代理的，如果实现接口的话是使用Jdk动态代理。那么传入的AdvisedSupport又是什么呢？其实就是一个生成代理对象所需要的信息的载体。我们贴出一部分属性看一下：

```java
public class AdvisedSupport extends ProxyConfig implements Advised {

    //以下几个都是ProxyConfig里的属性
    private boolean proxyTargetClass = false;

	private boolean optimize = false;

	boolean opaque = false;

	boolean exposeProxy = false;

	private boolean frozen = false;
}
```

这五个属性的详细情况如下：
+ proxyTargetClass，该属性为true，那么AopProxyFactory会使用Cglib对织入对象进行动态代理。
+ optimize，该属性主要用于告知代理对象是否需要采取进一步的优化措施。
+ opaque，该属性用于控制生成的代理对象是否可以强制转型为Advised，默认false代表可以，可以通过Advised查询对象的一些状态。
+ exposeProxy，设置该属性可以让AOP框架生成代理对象时，将当前代理对象绑定到ThreadLocal。如果目标对象需要访问当前代理对象，可以通过AopContext.currentProxy()获取。
+ frozen，该属性设置为true，则针对代理对象生成的配置信息完成就不允许改动，可以优化代理对象生成的性能。

