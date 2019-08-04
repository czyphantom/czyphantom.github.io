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

Spring的运行基础是应用上下文，即ApplicationContext，其典型实现类ClassPathXmlApplicationContext，继承链为ClassPathXmlApplicationContext -> AbstractXmlApplicationContext -> AbstractRefreshableApplicationContext -> AbstractApplicationContext -> DefaultResourceLoader -> ResourceLoader。最上层的ResourceLoader为资源加载器，负责查找和定位资源。ClassPathXmlApplicationContext通过以下构造器加载配置文件：

```java
    public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}
```

接着调用以下构造器，主要目的是设置资源加载和解析类，设置配置路径

```java
    public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent) throws BeansException {
		//该方法一直往上调用到AbstractApplicationContext的构造器，主要就是设置ResourcePatternResolver，使用的实例为PathMatchingResourcePatternResolver
		super(parent);
		//为ApplicationContext设置配置路径
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```

refresh方法是初始化应用上下文的核心，实现在AbstractApplicationContext里，代码如下：

```java
    public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			//预处理，进行必要的初始化和验证
			prepareRefresh();

			//取得BeanFactory，重点，实现在AbstractRefreshableApplicationContext
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			//准备BeanFacotry，设置类加载器等操作
			prepareBeanFactory(beanFactory);

			try {
				//对BeanFactory设置特殊的后置处理器，由需要的子类实现
				postProcessBeanFactory(beanFactory);

				//调用BeanFacotry后置处理器
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

				//实例化所有非延迟初始化的单例bean，也是重点
				finishBeanFactoryInitialization(beanFactory);

				//完成应用上下文的创建
				finishRefresh();
			}

			catch (BeansException ex) {
				//略
			}
		}
	}
```

从refresh方法我们可以看出，实例化ApplicationContext其实做了很多步骤：
1. 获取BeanFactory。
2. 对BeanFactory设置后置处理器。
3. 调用后置处理器。
4. 注册BeanPostProcessor。
5. 检查并注册监听Beans。
6. 实例化所有非延迟初始化的单例Bean。

## 构建BeanFactory

构建BeanFactory的方法由AbstractRefreshableApplicationContext实现，代码如下：

```java
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
         //创建新的BeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
         //主要定义是否允许bean定义覆盖，是否允许循环依赖（默认允许），构造方法的循环依赖是不允许的
			customizeBeanFactory(beanFactory);
         //使用beanFactory加载bean定义，重点
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

构建是BeanFactory的核心是加载bean定义，即方法loadBeanDefinitions,此方法调用AbstractXmlApplicationContext的loadBeanDefinitions方法，最终调用XmlBeanDefinitionReader中的doLoadBeanDefinitions方法,如下：

```java
	
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		initBeanDefinitionReader(beanDefinitionReader);
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

	//XmlBeanDefinitionReader
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}

	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		//省略

		try {
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		//省略catch和finally块
	}

		protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			//根据配置文件加载Document，可见Spring使用DOM方式加载
			Document doc = doLoadDocument(inputSource, resource);
			//注册Bean定义，重点
			return registerBeanDefinitions(doc, resource);
		}
		//省略catch块
	}
```

加载Bean定义主要根据配置文件抽象为资源，根据资源的inputstream使用DOM方式加载为Document，接着注册Bean定义。注册bean定义最终调用DefaultBeanDefinitionDocumentReader的registerBeanDefinitions方法，代码如下：

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
		//空实现，目的在于给子类一个把自定义的标签转为Spring标准标签的机会
		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
		//重点，解析根元素
		postProcessXml(root);
		this.delegate = parent;
	}    
```

注册Bean定义首先需要根据加载的XML文件形成的Document，并对其中标签进行解析，首先加载根元素解析，通过使用parseBeanDefinitions方法来解析根元素：

```java
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
                  //解析普通节点
						parseDefaultElement(ele, delegate);
					}
					else {
						//非默认命名空间的元素交由delegate处理
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

解析时，获取根节点的每一个子节点，调用parseDefaultElement方法解析普通节点，接着使用processBeanDefinition方法处理普通bean标签：

```java

	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
         //处理bean标签
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			doRegisterBeanDefinitions(ele);
		}
	}

    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
       //根据bean标签获取一个BeanDefinitionHolder，beanholder中包含了bean的名称和一些属性信息
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
               //注册bean定义
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

解析标签时，首先通过DOM节点得到了一个BeanDefinitionHolder（这个类后面会说明用处），接着调用BeanDefinitionReaderUtils.registerBeanDefinition方法来注册Bean定义：

```java
    public static void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) throws BeanDefinitionStoreException {
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
		//省略
        BeanDefinition oldBeanDefinition;
        //beanDefinitionMap是一个键是beanName，值是beanDefinition的ConcurrentHashMap，大小为256
        oldBeanDefinition = this.beanDefinitionMap.get(beanName);
        //查看是否已经注册，如果已经注册了，则将新的put进去
        if (oldBeanDefinition != null) {
            //省略
            this.beanDefinitionMap.put(beanName, beanDefinition);
        }
        else {
            //如果bean已经开始创建了，此时要保证线程安全
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
                //最终放进map实现注册
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
	public AbstractBeanDefinition parseBeanDefinitionElement(Element ele, String beanName, @Nullable BeanDefinition containingBean) {
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

构建完BeanFactory之后，还需要实例化bean（主要是作用域为Singleton的bean)，回顾refresh方法可以发现是finishBeanFactoryInitialization方法所完成的。在完成必要的设置之后，调用DefaultListableBeanFactory的preInstantiateSingletons方法来实例化：

```java
    public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}
      //得到所有beanName的名称
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		//重头戏，开始实例化了，实例化所有非延迟初始化的单例bean
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					//FactoryBean的处理逻辑
				}
				else {
					getBean(beanName);
				}
			}
		}
		//忽略实现SmarInitializingSingleton的Bean，回调
    }
```

实例化的方式实际上就是先取得所有bean的名称的List，然后根据beanName调用getBean实例化bean，它最后调用doGetBean方法,我们只看关键部分的代码：

```java
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		//检查下是否已经创建过了
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			//如果是普通bean，直接返回sharedInstance，否则返回FactoryBean创建的对象
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		} else {
      		// 检查一下这个BeanDefinition在容器中是否存在
      		BeanFactory parentBeanFactory = getParentBeanFactory();
      		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         	   //如果当前容器不存在这个BeanDefinition，试试父容器中有没有
        		   String nameToLookup = originalBeanName(name);
         		if (args != null) {
            		//返回父容器的查询结果
            		return (T) parentBeanFactory.getBean(nameToLookup, args);
         		} else {
            		return parentBeanFactory.getBean(nameToLookup, requiredType);
        		   }
      		}	
 
      		if (!typeCheckOnly) {
         		//typeCheckOnly为false，将当前beanName放入一个alreadyCreated的Set集合中。
         		markBeanAsCreated(beanName);
      		}
      		try {	
         		final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         		checkMergedBeanDefinition(mbd, beanName, args);
 
         		// 注意，这里的依赖指的是depends-on属性中定义的依赖，不是@Autowired的依赖
         		String[] dependsOn = mbd.getDependsOn();
         		if (dependsOn != null) {
            		for (String dep : dependsOn) {
               			//检查是不是有循环依赖，这里的循环依赖指的是depends-on的循环依赖
               			if (isDependent(beanName, dep)) {
                  			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        		"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               			}
               			//注册一下依赖关系
               			registerDependentBean(dep, beanName);
               			//先初始化被依赖项
               			getBean(dep);
            		}
         		}
 
         		// 创建 singleton 的实例
         		if (mbd.isSingleton()) {
            		sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
               			@Override
               			public Object getObject() throws BeansException {
                  			try {
                     			// 执行创建 Bean，详情后面再说
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
 
         		// 创建 prototype 的实例
         		else if (mbd.isPrototype()) {
            	//省略
         		}
         		else {
				   //如果不是 singleton 和 prototype 的话，需要委托给相应的实现类来处理
         		}
      		}
      		catch (BeansException ex) {
         		cleanupAfterBeanCreationFailure(beanName);
         	throw ex;
      	}
   	}
 
   //省略检查类型
   return (T) bean;
}
```

在getBean时，首先会检查是否该bean已经被创建过，如果有，直接返回。然后对该bean的depends-on属性的bean进行递归实例化，最后再实例化该bean。实例化bean的方法为 createBean,该方法最终调用AbstractAutowireCapableBeanFactory的doCreateBean方法，我们看看关键部分代码：

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
   if (logger.isDebugEnabled()) {
      logger.debug("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;
 
   //确保BeanDefinition中的Class被加载
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }
 
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }
 
   try {
      //让BeanPostProcessor在这一步有机会返回代理，而不是bean实例
      //使用@Autowire注解的bean在这里返回
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean; 
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }
   //创建bean，重点
   Object beanInstance = doCreateBean(beanName, mbdToUse, args);
   if (logger.isDebugEnabled()) {
      logger.debug("Finished creating instance of bean '" + beanName + "'");
   }
   return beanInstance;
}
```
注意到resolveBeforeInstantiation方法可以处理@Autowired注解的bean，这也是@Autowired注解的原理。我们来看看doCreateBean方法的实现：

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
      throws BeanCreationException {
 
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   //这就是目标的实例
   final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
   Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
   mbd.resolvedTargetType = beanType;
 
   // 省略一部分
  
   //解决循环依赖的问题
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isDebugEnabled()) {
         logger.debug("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      addSingletonFactory(beanName, new ObjectFactory<Object>() {
         @Override
         public Object getObject() throws BeansException {
            return getEarlyBeanReference(beanName, mbd, bean);
         }
      });
   }
 
   Object exposedObject = bean;
   try {
      //这一步负责属性装配
      populateBean(beanName, mbd, instanceWrapper);
      if (exposedObject != null) {
         //处理init-method等
         exposedObject = initializeBean(beanName, exposedObject, mbd);
      }
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }
 
	//省略一部分
    return exposedObject;
}
```

createBean时通过createBeanInstance创建实例，然后处理循环依赖，接着属性装配，然后再处理init-method。先看createBeanInstance方法：

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
   // 确保已经加载了此class
   Class<?> beanClass = resolveBeanClass(mbd, beanName);
 
   // 省略权限验证和工厂方法

   // 如果不是第一次创建，比如第二次创建prototype bean。
   // 这种情况下，我们可以从第一次创建知道，采用无参构造函数，还是构造函数依赖注入来完成实例化
   boolean resolved = false;
   boolean autowireNecessary = false;
   if (args == null) {
      synchronized (mbd.constructorArgumentLock) {
         if (mbd.resolvedConstructorOrFactoryMethod != null) {
            resolved = true;
            autowireNecessary = mbd.constructorArgumentsResolved;
         }
      }
   }
   if (resolved) {
      if (autowireNecessary) {
         // 构造函数依赖注入
         return autowireConstructor(beanName, mbd, null, null);
      }
      else {
         //无参构造函数
         return instantiateBean(beanName, mbd);
      }
   }
 
   //判断是否采用有参构造函数
   Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
   if (ctors != null ||
         mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
         mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
      //构造函数依赖注入
      return autowireConstructor(beanName, mbd, ctors, args);
   }
 
   //调用无参构造函数
   return instantiateBean(beanName, mbd);
}

//无参构造函数实例化
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
   try {
      Object beanInstance;
      final BeanFactory parent = this;
      if (System.getSecurityManager() != null) {
         beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
 
               return getInstantiationStrategy().instantiate(mbd, beanName, parent);
            }
         }, getAccessControlContext());
      }
      else {
         //实例化，getInstantiationStrategy方法获取一个实例化的策略
         beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
      }
      // 包装一下，返回
      BeanWrapper bw = new BeanWrapperImpl(beanInstance);
      initBeanWrapper(bw);
      return bw;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
   }
}

//实例化
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
 
   // 如果不存在方法覆写，那就使用java反射进行实例化，否则使用CGLIB
   if (bd.getMethodOverrides().isEmpty()) {
      Constructor<?> constructorToUse;
      synchronized (bd.constructorArgumentLock) {
         constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
         if (constructorToUse == null) {
            final Class<?> clazz = bd.getBeanClass();
            if (clazz.isInterface()) {
               throw new BeanInstantiationException(clazz, "Specified class is an interface");
            }
            try {
               if (System.getSecurityManager() != null) {
                  constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
                     @Override
                     public Constructor<?> run() throws Exception {
                        return clazz.getDeclaredConstructor((Class[]) null);
                     }
                  });
               }
               else {
                  constructorToUse = clazz.getDeclaredConstructor((Class[]) null);
               }
               bd.resolvedConstructorOrFactoryMethod = constructorToUse;
            }
            catch (Throwable ex) {
               throw new BeanInstantiationException(clazz, "No default constructor found", ex);
            }
         }
      }
      //利用构造方法进行实例化
      return BeanUtils.instantiateClass(constructorToUse);
   }
   else {
      //存在方法覆写，利用CGLIB来完成实例化，需要依赖于CGLIB生成子类
      return instantiateWithMethodInjection(bd, beanName, owner);
   }
}
```

创建bean实例时返回的不是bean的实例而是一个BeanWrapper，根据是否有有参构造器判断使用哪种构造器实例化，实例化时使用策略模式确定使用哪种实例化，如果没有方法覆写则使用反射实例化，否则使用CGLIB实例化。再看属性注入：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
   //获取bean的所有属性值
   PropertyValues pvs = mbd.getPropertyValues();
 
   if (bw == null) {
      if (!pvs.isEmpty()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
         return;
      }
   }
 
   // 到这步的时候，bean实例化完成（通过工厂方法或构造方法），但是还没开始属性设值，
   // InstantiationAwareBeanPostProcessor的实现类可以在这里对bean进行状态修改，
   boolean continueWithPropertyPopulation = true;
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 如果返回 false，代表不需要进行后续的属性设值，也不需要再经过其他的BeanPostProcessor的处理
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               continueWithPropertyPopulation = false;
               break;
            }
         }
      }
   }
 
   if (!continueWithPropertyPopulation) {
      return;
   }
 
   if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
         mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
 
      // 通过名字找到所有属性值，如果是bean依赖，先初始化依赖的bean。记录依赖关系
      if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
         autowireByName(beanName, mbd, bw, newPvs);
      }
 
      // 通过类型装配
      if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
         autowireByType(beanName, mbd, bw, newPvs);
      }
 
      pvs = newPvs;
   }
 
   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
   boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);
 
   if (hasInstAwareBpps || needsDepCheck) {
      PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      if (hasInstAwareBpps) {
         for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
               InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
               // 这里有个非常有用的 BeanPostProcessor 进到这里: AutowiredAnnotationBeanPostProcessor
               pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvs == null) {
                  return;
               }
            }
         }
      }
      if (needsDepCheck) {
         checkDependencies(beanName, mbd, filteredPds, pvs);
      }
   }
   // 设置bean实例的属性值
   applyPropertyValues(beanName, mbd, bw, pvs);
}

	protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		if (pvs.isEmpty()) {
			return;
		}

		if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
			((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
		}

		MutablePropertyValues mpvs = null;
		List<PropertyValue> original;

		if (pvs instanceof MutablePropertyValues) {
			mpvs = (MutablePropertyValues) pvs;
			if (mpvs.isConverted()) {
				// Shortcut: use the pre-converted values as-is.
				try {
					bw.setPropertyValues(mpvs);
					return;
				}
				catch (BeansException ex) {
					throw new BeanCreationException(
							mbd.getResourceDescription(), beanName, "Error setting property values", ex);
				}
			}
			original = mpvs.getPropertyValueList();
		}
		else {
			original = Arrays.asList(pvs.getPropertyValues());
		}

		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}
		BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

		// Create a deep copy, resolving any references for values.
		List<PropertyValue> deepCopy = new ArrayList<>(original.size());
		boolean resolveNecessary = false;
		for (PropertyValue pv : original) {
			if (pv.isConverted()) {
				deepCopy.add(pv);
			}
			else {
				String propertyName = pv.getName();
				Object originalValue = pv.getValue();
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				if (convertible) {
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}
				// Possibly store converted value in merged bean definition,
				// in order to avoid re-conversion for every created bean instance.
				if (resolvedValue == originalValue) {
					if (convertible) {
						pv.setConvertedValue(convertedValue);
					}
					deepCopy.add(pv);
				}
				else if (convertible && originalValue instanceof TypedStringValue &&
						!((TypedStringValue) originalValue).isDynamic() &&
						!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
					pv.setConvertedValue(convertedValue);
					deepCopy.add(pv);
				}
				else {
					resolveNecessary = true;
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
		if (mpvs != null && !resolveNecessary) {
			mpvs.setConverted();
		}

		// Set our (possibly massaged) deep copy.
		try {
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Error setting property values", ex);
		}
	}
```

属性注入时，首先进行一些PostProcessor操作，接着判断依赖的属性是否为bean，如果是bean，还需要先初始化依赖的bean，然后处理所有的BeanProcessor，最后再装配属性，属性的装配通过BeanWrapper的PropertyValue属性装配。