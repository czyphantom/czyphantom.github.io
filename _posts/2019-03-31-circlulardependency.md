---
layout:     post
title:      Spring源码分析（二）
subtitle:   循环依赖
date:       2019-03-31
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Spring
---

循环依赖问题指的是两个或两个以上的bean互相持有对方，最终形成闭环。Spring有两种循环依赖场景：一种是构造器的循环依赖，此种方式提前抛出异常；另一种是属性的循环依赖，此种通过缓存的方式可以解决。Spring的循环依赖的理论依据其实是基于Java的引用传递，当我们获取到对象的引用时，对象的field或则属性是可以延后设置的(但是构造器必须是在获取引用之前)，因此构造器的循环依赖无法解决。

在上一篇文章里我们介绍了Spring实例化Bean的方法是doGetBean，在实例化单例bean时，首先会从缓存中取单例的bean，调用的是getSingleton方法：

```java
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        //从一级缓存取，取不到且bean在创建中，从二级缓存取
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
                //二级缓存取不到，且允许提前暴露引用，从三级缓存取，同时加入到二级缓存中，并从三级缓存中移除
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
```

一个一个的解释，首先缓存分为三级，分别为：
+ singletonObjects：单例对象的cache，这种是已经创建好了。
+ earlySingletonObjects：提前曝光的单例对象的Cache。
+ singletonFactories：单例对象工厂的cache。

这三个在DefaultSingletonBeanRegistry中都有定义：

```java
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);

	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

	private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
```

实例对象又是在何时加入到缓存里的呢？我们发现在AbstractBeanFacotry里的doGetBean有这样几行：

```java
	if (mbd.isSingleton()) {
        //这一步提前获取bean
		sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
				 @Override
				public Object getObject() throws BeansException {
					try {
						return createBean(beanName, mbd, args);
                    }
					catch (BeansException ex) {
						destroySingleton(beanName);
						throw ex;
					}
				}
			});
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
	}
```

看一下getSingleton方法做了什么：

```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "'beanName' must not be null");
		synchronized (this.singletonObjects) {
            //先从一级缓存取，取不出来，则从singletonFactory取
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				//省略一部分
				try {
                    //这一步能取到bean的实例了，其保证在于singletonFactory实现的getObject方法，上文可知这是由createBean方法保证，稍后再叙述
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				//省略catch块和finally块
				if (newSingleton) {
					//加入实例
					addSingleton(beanName, singletonObject);
				}
			}
			return (singletonObject != NULL_OBJECT ? singletonObject : null);
		}
	}

    protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
            //加入到一级缓存，并从二级、三级缓存中移除
			this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
```

getSingleton方法首先从一级缓存中取，如果bean已经装配完就可以取到了，否则通过objectFactory缓存取，这一步能取到主要由于objectFactory类的getObject方法需要提供获取bean的过程，这里通过createbean方法将bean加入到ObjectFactory。再来看createBean方法如何将bean加入到ObjectFactory，其最后调用的是doCreateBean方法，代码如下：

```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
		//省略一部分
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			//addSingletonFactory将beanName加入到SingletonFactory
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
		}
		//省略一部分
	}

	protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				this.singletonFactories.put(beanName, singletonFactory);
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
	}
	
	//实际上可以看到是需要满足hasInstantiationAwareBeanPostProcessors()，即一些特殊的注解如@Autowired才会把引用暴露
	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
					if (exposedObject == null) {
						return exposedObject;
					}
				}
			}
		}
		return exposedObject;
	}
```

createBean方法创建完实例后，通过将引用提前暴露给ObjectFactory，保证在获取bean实例时，可以从ObjectFactory中获取。

# 总结

有了缓存，就可以解决循环依赖的问题了，举个例子，例如A的某个field依赖了B的实例对象，同时B的某个field依赖了A的实例对象。首先A完成了初始化的第一步，并且将自己提前曝光到singletonFactories中，此时进行初始化的第二步，发现自己依赖对象B，此时就尝试去get(B)，发现B还没有被create，所以走create流程，B在初始化第一步的时候发现自己依赖了对象A，于是尝试get(A)，尝试一级缓存singletonObjects(肯定没有，因为A还没初始化完全)，尝试二级缓存earlySingletonObjects（也没有），尝试三级缓存singletonFactories，由于A通过ObjectFactory将自己提前曝光了，所以B能够通过ObjectFactory.getObject拿到A对象(虽然A还没有初始化完全，但是总比没有好)，B拿到A对象后顺利完成了初始化阶段1、2、3，完全初始化之后将自己放入到一级缓存singletonObjects中。此时返回A中，A此时能拿到B的对象顺利完成自己的初始化阶段2、3，最终A也完成了初始化，进去了一级缓存singletonObjects中，而且更加幸运的是，由于B拿到了A的对象引用，所以B现在hold住的A对象完成了初始化。