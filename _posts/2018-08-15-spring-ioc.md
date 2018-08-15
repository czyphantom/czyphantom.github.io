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

refresh方法的核心在于构建BeanFactory，主要由AbstractRefreshableApplicationContext实现，代码如下：

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

比较重要的是BeanFactory加载Bean定义的方法,最终调用XmlBeanDefinitionReader中的doLoadBeanDefinitions方法,如下：

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

该方法首先加载配置文件的Document，接着注册Bean定义。注册Bean定义最终调用DefaultBeanDefinitionDocumentReader的registerBeanDefinitions方法获得Document的根元素，然后调用doRegisterBeanDefinitions方法注册根元素，代码如下：

```java
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
		Element root = doc.getDocumentElement();
		doRegisterBeanDefinitions(root);
	}

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

核心在于parseDefaultElement方法解析普通节点，该方法使用processBeanDefinition方法处理普通bean标签，代码如下：

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

该方法会调用BeanDefinitionReaderUtils.registerBeanDefinition方法来注册Bean定义，实现如下：

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

最后通过registerBeanDefinition正式注册：

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
                // 最终放进这个map 实现注册
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

将beanName和beanDefinition放进一个ConcurrentHashMap中。

bean的定义是在DefaultBeanDefinitionDocumentReader.processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate)方法中创建的，在这里创建了BeanDefinitionHolder，而该实例中解析Bean并将Bean保存在该对象中，所以称为持有者。该实例会调用parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean)方法，该方法用于解析XML文件并创建一个BeanDefinitionHolder返回，该方法会调用 parseBeanDefinitionElement(ele, beanName, containingBean)方法，方法如下：

```java
    public AbstractBeanDefinition parseBeanDefinitionElement(
            Element ele, String beanName, @Nullable BeanDefinition containingBean) {

        this.parseState.push(new BeanEntry(beanName));

        String className = null;
        if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
            className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
        }
        String parent = null;
        if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
            parent = ele.getAttribute(PARENT_ATTRIBUTE);
        }

        try {
            AbstractBeanDefinition bd = createBeanDefinition(className, parent);

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

该方法很简单，创建一个Definition的持有者，然后设置该持有者的Class对象，该对象就是我们在配置文件中配置的Class对象。最后返回。到这里，我们一走完了第一步，创建bean工厂，生成Bean定义。