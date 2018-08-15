---
layout:     post
title:      Spring源码阅读（一）
subtitle:   IoC源码
date:       2018-08-15
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Spring
---

# ClassPathXmlApplicationContext

Spring的运行基础是应用上下文，即ApplicationContext，其典型实现类ClassPathXmlApplicationContext，继承链为ClassPathXmlApplicationContext -> AbstractXmlApplicationContext -> AbstractRefreshableApplicationContext -> AbstractApplicationContext。在调用以下构造器时：

```java
    public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}
```

通过该构造器加载配置文件，该构造器又调用：

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

在完成必要的设置后，该构造器又调用refresh方法，refresh方法是初始化的核心，代码如下：

```java
    public void refresh() throws BeansException, IllegalStateException {
        //可能有多个应用上下文环境，所以要保证线程安全
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

				//初始化其他特殊的Beans，由子类实现
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

该方法的核心在于构建BeanFactory，最终由AbstractRefreshableApplicationContext实现，代码如下：

```java
	protected final void refreshBeanFactory() throws BeansException {
        //如果已经有BeanFactory了，说明有其他线程创建了，销毁所有的单例Bean,关闭此次的BeanFactory
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

创建BeanFactory考虑到了多线程的情况，因此需要加一些判定。另外比较重要的是BeanFactory加载Bean定义,该方法的实现在AbstractXmlApplicationContext：

```java
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		//根据BeanFactory准备解析XML中的Bean定义了
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		//进行一些必要的设置
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		initBeanDefinitionReader(beanDefinitionReader);
        //该方法加载Bean定义
		loadBeanDefinitions(beanDefinitionReader);
	}

	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}

    public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int counter = 0;
		for (Resource resource : resources) {
			counter += loadBeanDefinitions(resource);
		}
		return counter;
	}   
```

loadBeanDefinitions的核心代码是XmlBeanDefinitionReader中的doLoadBeanDefinitions，如下：

```java
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
            //根据配置文件加载Document
			Document doc = doLoadDocument(inputSource, resource);
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		//省略其他捕获异常的部分
	}
```

该方法首先加载文件的Document，接着注册。注册Bean定义的相关代码如下：

```java
    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		documentReader.setEnvironment(getEnvironment());
		int countBefore = getRegistry().getBeanDefinitionCount();
        //该方法调用DefaultBeanDefinitionDocumentReader的registerBeanDefinitions方法获得Document根元素后，再调用doRegisterBeanDefinitions方法解析XML文档
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

获得Document的根元素后，使用parseBeanDefinitions来解析：

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

解析的核心在于parseDefaultElement方法，该方法使用processBeanDefinition方法处理bean标签，代码如下：

```java
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);// 解析
        if (bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
                // Register the final decorated instance.
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
            }
            catch (BeanDefinitionStoreException ex) {
                getReaderContext().error("Failed to register bean definition with name '" +
                        bdHolder.getBeanName() + "'", ele, ex);
            }
            // Send registration event.
            getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }
    }
```

首先创建一个BeanDefinitionHolder，该方法会调用BeanDefinitionReaderUtils.registerBeanDefinition方法,最后执行容器通知事件。该静态方法实现如下：

```java
    public static void registerBeanDefinition(
            BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
            throws BeanDefinitionStoreException {

        // Register bean definition under primary name.
        String beanName = definitionHolder.getBeanName();
        registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());// 注册

        // Register aliases for bean name, if any.
        String[] aliases = definitionHolder.getAliases();
        if (aliases != null) {
            for (String alias : aliases) {
                registry.registerAlias(beanName, alias);
            }
        }
    }
```

可以看到首先从bean的持有者那里获取了beanName，然后调用registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition())， 将bean的名字和BeanDefinition注册，最后：

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

        oldBeanDefinition = this.beanDefinitionMap.get(beanName);
        if (oldBeanDefinition != null) {
            if (!isAllowBeanDefinitionOverriding()) {
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                        "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
                        "': There is already [" + oldBeanDefinition + "] bound.");
            }
            else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
                // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
                            "' with a framework-generated bean definition: replacing [" +
                            oldBeanDefinition + "] with [" + beanDefinition + "]");
                }
            }
            else if (!beanDefinition.equals(oldBeanDefinition)) {
                if (this.logger.isInfoEnabled()) {
                    this.logger.info("Overriding bean definition for bean '" + beanName +
                            "' with a different definition: replacing [" + oldBeanDefinition +
                            "] with [" + beanDefinition + "]");
                }
            }
            else {
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Overriding bean definition for bean '" + beanName +
                            "' with an equivalent definition: replacing [" + oldBeanDefinition +
                            "] with [" + beanDefinition + "]");
                }
            }
            this.beanDefinitionMap.put(beanName, beanDefinition);
        }
        else {
            if (hasBeanCreationStarted()) {
                // Cannot modify startup-time collection elements anymore (for stable iteration)
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
                // Still in startup registration phase // 最终放进这个map 实现注册
                this.beanDefinitionMap.put(beanName, beanDefinition);// 走这里 // private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
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

将beanName和beanDefinition放进一个ConcurrentHashMap中。

那么这个 beanDefinition 是时候创建的呢？ 就是在 DefaultBeanDefinitionDocumentReader.processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) 方法中，在这里创建了 BeanDefinitionHolder， 而该实例中解析Bean并将Bean 保存在该对象中。所以称为持有者。该实例会调用 parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) 方法，该方法用于解析XML文件并创建一个 BeanDefinitionHolder 返回，该方法会调用 parseBeanDefinitionElement(ele, beanName, containingBean) 方法， 我们看看该方法：

```java
    public AbstractBeanDefinition parseBeanDefinitionElement(
            Element ele, String beanName, @Nullable BeanDefinition containingBean) {

        this.parseState.push(new BeanEntry(beanName));

        String className = null;
        if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
            className = ele.getAttribute(CLASS_ATTRIBUTE).trim();// 类全限定名称
        }
        String parent = null;
        if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
            parent = ele.getAttribute(PARENT_ATTRIBUTE);
        }

        try {
            AbstractBeanDefinition bd = createBeanDefinition(className, parent);// 创建

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
        catch (ClassNotFoundException ex) {
            error("Bean class [" + className + "] not found", ele, ex);
        }
        catch (NoClassDefFoundError err) {
            error("Class that bean class [" + className + "] depends on not found", ele, err);
        }
        catch (Throwable ex) {
            error("Unexpected failure during bean definition parsing", ele, ex);
        }
        finally {
            this.parseState.pop();
        }

        return null;
    }
```

可以看到，该方法从XML元素中取出 class 元素，然后拿着className调用 createBeanDefinition(className, parent) 方法，该方法核心是调用 BeanDefinitionReaderUtils.createBeanDefinition 方法，我们看看该方法：

```java
    public static AbstractBeanDefinition createBeanDefinition(
            @Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {

        GenericBeanDefinition bd = new GenericBeanDefinition();// 泛型的bean定义，也就是最终生成的bean定义。
        bd.setParentName(parentName);
        if (className != null) {
            if (classLoader != null) {
                bd.setBeanClass(ClassUtils.forName(className, classLoader));// 设置Class 对象
            }
            else {
                bd.setBeanClassName(className);
            }
        }
        return bd;
    }
```

该方法很简单，创建一个Definition的持有者，然后设置该持有者的Class对象，该对象就是我们在配置文件中配置的Class对象。最后返回。到这里，我们一走完了第一步，创建bean工厂，生成Bean定义。