---
layout:     post
title:      Spring Boot源码阅读
subtitle:   Spring Boot启动原理
date:       2019-04-15
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - Spring
---

一个SpringBoot应用的启动类通常如下所示：

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

这里只有两个地方需要注意，一个是@SpringBootApplication注解，另一个是SpringApplication.run()方法，这两个东西就是SpringBoot应用启动的关键。首先看一下@SpringBootApplication注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    //省略
}
```

除了几个Java自带的注解外，@SpringBootApplication注解有三个其他的注解：@SpringBootConfiguration（点进去发现其实也是@Configuration），@EnableAutoConfiguration以及@ComponentScan。@Configuration注解的类都是一个JavaConfig的配置类，可以在@Configuration注解的类里使用@Bean将bean注册到IoC容器中，启动类被标注了@Configuration意味着启动类也是一个JavaConfig类。@ComponentScan自动扫描并加载符合条件的组件或者bean定义，最终将这些bean定义加载到IoC容器中，**由于默认扫描的是有该注解的类所在的package，所以一般将启动类放在根目录下**。**@EnableAutoConfiguration负责自动配置，这时Spring Boot的核心之一**，我们看它有哪些定义：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    //省略
}
```

@AutoConfigurationPackage注解负责自动配置包，主要通过Registrar注册了一个bean定义，将启动类所在包注册到IoC容器中，**@Import负责导入自动配置的组件，通过AutoConfigurationImportSelector将所有符合条件的@Configuration配置都加载到当前Spring Boot创建并使用的IoC容器**，AutoConfigurationImportSelector实现了BeanFactoryAware接口，熟知Bean的生命周期的的都知道BeanFatoryAware接口的Bean会将BeanFactory注入到Bean中。

Rigstrar代码如下：

```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

        @Override
        public void registerBeanDefinitions(AnnotationMetadata metadata,
                BeanDefinitionRegistry registry) {
			//PackageImport(metadata).getPackageName()实际返回了当前主程序类的同级以及子级的包组件
            register(registry, new PackageImport(metadata).getPackageName());
        }
}
```

AutoConfigurationImportSelector有一个selectImports方法，selectImports方法返回一组bean，@EnableAutoConfiguration注解借助@Import注解将这组bean注入到spring容器中，springboot正式通过这种机制来完成bean的注入的。：

```java
	@Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
                .loadMetadata(this.beanClassLoader);
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
                attributes);
        configurations = removeDuplicates(configurations);
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        configurations = filter(configurations, autoConfigurationMetadata);
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return StringUtils.toStringArray(configurations);
    }

	protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
		//这一步从所有的jar包中读取META-INF/spring.factories文件信息
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
				getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
		Assert.notEmpty(configurations,
				"No auto configuration classes found in META-INF/spring.factories. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```

看一下SpringFacotriesLoader的loadFactoryNames方法：

```java
	public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
	}

	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
			//获取META-INF/spring.factories
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					List<String> factoryClassNames = Arrays.asList(
							StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
					result.addAll((String) entry.getKey(), factoryClassNames);
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```

META-INF/spring.factories文件定义了需要自动配置的bean，通过读取这个配置获取一组@Configuration类。每个xxxAutoConfiguration都是一个基于java的bean配置类。实际上，这些xxxAutoConfiguratio不是所有都会被加载，会根据xxxAutoConfiguration上的@ConditionalOnClass等条件判断是否加载。通过反射机制将spring.factories中@Configuration类实例化为对应的java实列。

以上为Spring Boot的自动配置原理，我们可以将自动配置的关键几步以及相应的注解总结如下：
1. @Configuration&与@Bean->基于java代码的bean配置
2. @Conditional->设置自动配置条件依赖
3. @EnableConfigurationProperties与@ConfigurationProperties->读取配置文件转换为bean。
4. @EnableAutoConfiguration、@AutoConfigurationPackage 与@Import->实现bean发现与加载。

接下来看SpringApplication的run方法：

```java
	public static ConfigurableApplicationContext run(Class<?> primarySource,
			String... args) {
		return run(new Class<?>[] { primarySource }, args);
	}

    public static ConfigurableApplicationContext run(Class<?>[] primarySources,
			String[] args) {
        //可见需要先实例化一个SpringApplication再调用run方法
		return new SpringApplication(primarySources).run(args);
	}

	public SpringApplication(Class<?>... primarySources) {
		this(null, primarySources);
	}

    //实际调用的构造器
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        //判断是否为Web程序
		this.webApplicationType = deduceWebApplicationType();
        //设置应用程序初始化器ApplicationContextInitializer，初始化器从spring.factories中读取配置文件key为org.springframework.context.ApplicationContextInitializer的value
		//并在Spring上下文被刷新之前进行初始化的操作。典型地比如在Web应用中，注册Property Sources或者是激活Profiles
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
        //设置应用程序事件监听器ApplicationListener，监听器也是从spring.factories文件中读取，监听感兴趣的事件
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        //找出启动类，设置到mainApplicationClass中
		this.mainApplicationClass = deduceMainApplicationClass();
	}

	//找启动类
	private Class<?> deduceMainApplicationClass() {
		try {
			//通过构造一个运行时异常，通过异常栈中方法名为main的栈帧来得到入口类的名字
			StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
			for (StackTraceElement stackTraceElement : stackTrace) {
				if ("main".equals(stackTraceElement.getMethodName())) {
					return Class.forName(stackTraceElement.getClassName());
				}
			}
		}
		catch (ClassNotFoundException ex) {
			// Swallow and continue
		}
		return null;
	}

    //运行的方法
    public ConfigurableApplicationContext run(String... args) {
        //创建一个秒表，开始计时
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
        //发出开始执行的事件
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			//准备环境
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			//打印艺术字
			Banner printedBanner = printBanner(environment);
            //这一步就是创建IoC容器了，后面和Spring的IoC应该类似
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			//上下文前置处理
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			//刷新上下文
			refreshContext(context);
			//上下文后置处理
			afterRefresh(context, applicationArguments);
            //容器启动完，停止计时
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
            //所有监听器运行IoC容器
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}

```

简单总结一下SpringBootApplication的run方法：其中主要有一个SpringApplicationRunListeners的概念，它作为Spring Boot容器初始化时各阶段事件的中转器，将事件派发给感兴趣的Listeners(在SpringApplication实例的构建过程中得到的)。这些阶段性事件将容器的初始化过程给构造起来，提供了比较强大的可扩展性。