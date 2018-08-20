---
layout:     post
title:      Spring源码阅读（一）
subtitle:   IOC
date:       2018-08-15
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Spring
---

Spring的运行基础是应用上下文，即ApplicationContext，其典型实现类ClassPathXmlApplicationContext，继承链为ClassPathXmlApplicationContext -> AbstractXmlApplicationContext -> AbstractRefreshableApplicationContext -> AbstractApplicationContext。通过以下构造器加载配置文件：

```java
    public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}
```

该构造器又调用以下构造器：

```java
    public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh,        ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```

在完成必要的设置后，该构造器又调用refresh方法，refresh方法是初始化应用上下文的核心，代码如下：

```java
    public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			//做一些预处理
			prepareRefresh();

			//获得BeanFactory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			//对BeanFactory先做一些预处理
			prepareBeanFactory(beanFactory);

			try {
				//对BeanFactory设置特殊的后置处理器，由需要的子类实现
				postProcessBeanFactory(beanFactory);

				//调用后置处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				//注册bean的后置处理器
				registerBeanPostProcessors(beanFactory);

				//初始化信息源
				initMessageSource();

				//初始化上下文的事件机制
				initApplicationEventMulticaster();

				//初始化其他特殊的beans，由子类实现
				onRefresh();

				//检查并注册监听Beans
				registerListeners();

				//实例化所有非延迟初始化的单例bean
				finishBeanFactoryInitialization(beanFactory);

				//完成应用上下文的创建
				finishRefresh();
			}

			catch (BeansException ex) {
				logger.warn("Exception encountered during context initialization - cancelling refresh attempt", ex);

				destroyBeans();

				cancelRefresh(ex);

				throw ex;
			}
		}
	}
```

# 构建BeanFactory

refresh方法的核心在于构建BeanFactory，主要由AbstractRefreshableApplicationContext实现，代码如下：

```java
	protected final void refreshBeanFactory() throws BeansException {
        //如果已经有BeanFactory了，说明有其他线程创建了，销毁所有的单例bean,关闭此次的BeanFactory
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
            //此时创建新的BeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
            //定制化BeanFactory
			customizeBeanFactory(beanFactory);
            //使用beanFactory加载bean定义
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

该方法比较重要的是BeanFactory加载bean定义的方法loadBeanDefinitions,此方法最终调用XmlBeanDefinitionReader中的doLoadBeanDefinitions方法,如下：

```java
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
            //根据配置文件加载Document，由此可见Spring使用的是DOM方式解析XML文件
			Document doc = doLoadDocument(inputSource, resource);
            //注册Bean定义
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		//省略其他捕获异常的部分
	}
```

该方法首先加载配置文件的Document，接着注册bean定义。注册bean定义最终调用DefaultBeanDefinitionDocumentReader的registerBeanDefinitions方法，此方法首先获得Document的根元素，接着调用doRegisterBeanDefinitions注册，注册的方法代码如下：

```java
	protected void doRegisterBeanDefinitions(Element root) {
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					return;
				}
			}
		}

		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}    
```

通过使用parseBeanDefinitions方法来解析根元素：

```java
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
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
			delegate.parseCustomElement(root);
		}
	}    
```

此方法获得根节点的每一个子节点后，调用parseDefaultElement方法解析普通节点，接着使用processBeanDefinition方法处理普通bean标签：

```java
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        //解析节点
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
            
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
            }
            catch (BeanDefinitionStoreException ex) {
                getReaderContext().error("Failed to register bean definition with name '" +
                        bdHolder.getBeanName() + "'", ele, ex);
            }

            getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }
    }
```

该方法首先通过DOM节点得到了一个BeanDefinitionHolder（这个类后面会说明用处），接着调用BeanDefinitionReaderUtils.registerBeanDefinition方法来注册Bean定义：

```java
    public static void registerBeanDefinition(
            BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
            throws BeanDefinitionStoreException {

        //获得bean的名称
        String beanName = definitionHolder.getBeanName();
        //这一步注册bean定义
        registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

        String[] aliases = definitionHolder.getAliases();
        if (aliases != null) {
            for (String alias : aliases) {
                registry.registerAlias(beanName, alias);
            }
        }
    }
```

最后通过DefaultListableBeanFactory的registerBeanDefinition方法正式注册：

```java
    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException {

        Assert.hasText(beanName, "Bean name must not be empty");
        Assert.notNull(beanDefinition, "BeanDefinition must not be null");

        if (beanDefinition instanceof AbstractBeanDefinition) {
            try {
                ((AbstractBeanDefinition) beanDefinition).validate();
            }
            catch (BeanDefinitionValidationException ex) {
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                        "Validation of bean definition failed", ex);
            }
        }

        BeanDefinition oldBeanDefinition;

        //beanDefinitionMap是一个键是beanName，值是beanDefinition的ConcurrentHashMap，大小为256
        oldBeanDefinition = this.beanDefinitionMap.get(beanName);
        if (oldBeanDefinition != null) {
            //省略一部分判定打日志的代码
            this.beanDefinitionMap.put(beanName, beanDefinition);
        }
        else {
            //如果bean已经开始创建了，需要保证线程安全
            //该方法主要通过让beanName绑定到ThreadLocal里判断是否为空以此来判断是否开始
            if (hasBeanCreationStarted()) {
                synchronized (this.beanDefinitionMap) {
                    this.beanDefinitionMap.put(beanName, beanDefinition);
                    List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                    updatedDefinitions.addAll(this.beanDefinitionNames);
                    updatedDefinitions.add(beanName);
                    this.beanDefinitionNames = updatedDefinitions;
                    if (this.manualSingletonNames.contains(beanName)) {
                        Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
                        updatedSingletons.remove(beanName);
                        this.manualSingletonNames = updatedSingletons;
                    }
                }
            }
            else {
                //最终放进map 实现注册
                this.beanDefinitionMap.put(beanName, beanDefinition);
                this.beanDefinitionNames.add(beanName);
                this.manualSingletonNames.remove(beanName);
            }
            this.frozenBeanDefinitionNames = null;
        }

        if (oldBeanDefinition != null || containsSingleton(beanName)) {
            resetBeanDefinition(beanName);
        }
    }
```

注册的最终目标就是将beanName和beanDefinition放进BeanFactory的ConcurrentHashMap中。

# 注册BeanDefinition

之前说过BeanDefinitionHolder，实际上在解析DOM节点时，就是通过BeanDefinitionHolder来生成bean定义并放入到该类中的。具体来说，是在BeanDefinitionParserDelgate类中的parseBeanDefinitionElement方法。该方法用于解析XML文件并创建一个BeanDefinitionHolder返回，如下所示：

```java
    @Nullable
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

        //将含有beanName的Entry推入一个LinkedList
		this.parseState.push(new BeanEntry(beanName));

        //以下获取DOM节点的class标签和parent标签
		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
            //重点，创建bean定义
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);

            //以下处理bean标签下的其他内容，省略不看了
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

			parseMetaElements(ele, bd);
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

			parseConstructorArgElements(ele, bd);
			parsePropertyElements(ele, bd);
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		//省略catch异常
		finally {
			this.parseState.pop();
		}

		return null;
	}
```

该方法将beanName推入一个LinkedList,接着根据DOM节点获取其parent和class子节点。通过这两个节点调用createBeanDefinition创建Bean定义：

```java
    public static AbstractBeanDefinition createBeanDefinition(
            @Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {

        GenericBeanDefinition bd = new GenericBeanDefinition();
        bd.setParentName(parentName);
        if (className != null) {
            if (classLoader != null) {
                bd.setBeanClass(ClassUtils.forName(className, classLoader));
            }
            else {
                bd.setBeanClassName(className);
            }
        }
        return bd;
    }
```

该方法很简单，创建一个bean定义，设置一下parent名称，bean的class或者class的名称即可，这样以来就得到一个完整的bean定义。算上之前的部分，一个beanFactory就被构建出来了，但是还需要将bean实例化。

# 实例化bean以及完成IoC

构建完BeanFactory之后，还需要实例化bean（主要是作用域位Singleton的bean)，回顾refresh方法可以发现是finishBeanFactoryInitialization方法所完成的。在完成必要的设置之后，调用preInstantiateSingletons方法来实例化：

```java
    public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

        //得到所有beanName的名称
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		//重头戏，开始实例化了，实例化所有非延迟初始化的单例bean
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
                    //getBean方法是实例化的核心
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}
    }
```

实例化的方式实际上就是先取得所有bean的名称，让后循环实例化bean，其中getBean方法是实例化的核心，它最后调用doGetBean方法,我们只看关键部分的代码：

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		Object sharedInstance = getSingleton(beanName);

        //省略一些检查操作

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

                //获得所有的依赖
				String[] dependsOn = mbd.getDependsOn();                
				if (dependsOn != null) {
                    //对每一个依赖的单例bean进行实例化
					for (String dep : dependsOn) {
                        //该方法判断循环依赖
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
                        //把依赖的bean注册到被依赖的bean
                        //实际上bean和它的依赖是放到一个ConcurrentHashMap里，键是bean,值是被依赖的bean。
						registerDependentBean(dep, beanName);
						try {
                            //递归调用getBean方法，关键
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				//正式实例化单例bean
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				//省略其他作用域的部分
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		//省略后置的一些检查

		return (T) bean;
	}
```

实例化的核心就是处理bean和它所依赖的bean，这里Spring首先得到bean所依赖的bean后，循环遍历这些bean，如果不存在循环依赖，递归调用getBean方法，循环之外将bean实例化，这样保证了最后才实例化最上层的bean。有两个个比较重要的方法，就是如何检测循环依赖的isDependent方法和创建实例的方法createBean方法，先看isDependent方法，isDependent方法最后调用DefaultSingletonBeanRegistry的isDependent方法：

```java
    private boolean isDependent(String beanName, String dependentBeanName, @Nullable Set<String> alreadySeen) {
        //这一步说明又递归到beanName
		if (alreadySeen != null && alreadySeen.contains(beanName)) {
			return false;
		}
        //处理重命名的，忽略不看
		String canonicalName = canonicalName(beanName);
        //获得当前bean的依赖bean
		Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
		if (dependentBeans == null) {
			return false;
		}
        //注意每次注册才会包含该bean，所以如果已经包含了，说明就有循环依赖了
		if (dependentBeans.contains(dependentBeanName)) {
			return true;
		}
		for (String transitiveDependency : dependentBeans) {
			if (alreadySeen == null) {
				alreadySeen = new HashSet<>();
			}
			alreadySeen.add(beanName);
            //递归判断
			if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
				return true;
			}
		}
		return false;
	}
```

**对于一个bean，它直接依赖的bean是可以直接得到的，但是在处理这些依赖时，每次需要注册之后才会在dependentBeanMap里出现，如果没有循环依赖，每个bean都只会在dependentBeanMap里出现，如果在处理依赖的bean时发现已经存在了，说明出现了循环依赖。**isDependent的主要思路就是在现有的已注册的bean中递归判断是否出现上述的情况。

再看实例化bean的方法createBean,该方法最终调用AbstractAutowireCapableBeanFactory的doCreateBean方法，我们看看关键部分代码：

```java
        //省略

        //实例化
        if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}

        //省略

        //初始化
        try {
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}

        //省略
```

如果是无参构造器构造，最后会通过反射构造出实例来，具体就不详细去看了。实例构造好了，还需要按意愿初始化。代码比较长也就不细看了，简言之就是读取配置的property属性来设置值，如果有依赖的bean的话就会递归调用getBean方法。

# 总结

在创建应用上下文的过程中，核心的思路就在于：
1. 加载配置文件并创建bean定义。
2. 将bean定义注册到BeanFactory。
3. 实例化所有单例非延迟加载的bean，并完成依赖注入。