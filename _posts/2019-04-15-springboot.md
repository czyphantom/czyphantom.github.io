---
layout:     post
title:      SpringBoot源码阅读
subtitle:   SpringBoot启动原理
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

除了几个Java自带的注解外，@SpringBootApplication注解有三个其他的注解：@SpringBootConfiguration（点进去发现其实也是@Configuration），@EnableAutoConfiguration以及@ComponentScan。@Configuration注解的类都是一个JavaConfig的配置类，可以在@Configuration注解的类里使用@Bean将bean注册到IoC容器中，启动类被标注了@Configuration意味着启动类也是一个JavaConfig类。@ComponentScan自动扫描并加载符合条件的组件或者bean定义，最终将这些bean定义加载到IoC容器中，**由于默认扫描的是有该注解的类所在的package，所以一般将启动类放在根目录下**。@EnableAutoConfiguration负责自动配置，我们看它有哪些定义：

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

@AutoConfigurationPackage注解负责自动配置包，主要通过Registrar注册了一个bean定义，将启动类所在包注册到IoC容器中，@Import负责导入自动配置的组件，通过AutoConfigurationImportSelector将所有符合条件的@Configuration配置都加载到当前SpringBoot创建并使用的IoC容器。

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
        //设置应用程序初始化器ApplicationContextInitializer，做一些初始化的工作
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
        //设置应用程序事件监听器ApplicationListener
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        //找出启动类，设置到mainApplicationClass中
		this.mainApplicationClass = deduceMainApplicationClass();
	}

    //运行的方法
    public ConfigurableApplicationContext run(String... args) {
        //创建一个秒表，开始计时
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
        //运行所有监听器
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
            //这一步就是创建IoC容器了，后面和Spring的IoC应该类似
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
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

简单总结一下SpringBootApplication的run方法：
+ 首先需要创建SpringBootApplication的实例，创建实例的过程中需要检查Web环境，设置初始化器和监听器。
+ 运行时首先启动一个计时器，接着需要启动所有的监听器。
+ 然后创建IoC容器。
+ 通过监听器运行IoC容器。猜测使用Tomcat或者Jetty运行SpringBoot应用也是通过监听器来的？