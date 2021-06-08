---
layout:     post
title:      "SpringBoot(一) 启动与自动配置"
date:       2018-10-05
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - springBoot
--- 

<font id="last-updated">最后更新于：2018-10-08</font>

[SpringBoot(一) 启动与自动配置](https://zhouj000.github.io/2018/10/05/springboot-1/)  
[SpringBoot(二) starter与servlet容器](https://zhouj000.github.io/2018/10/10/springboot-2/)  
[SpringBoot(三) Environment](https://zhouj000.github.io/2019/10/08/springboot-3/)  
[SpringBoot(四) 集成apollo遇到的事儿](https://zhouj000.github.io/2019/10/16/springboot-4/)  
[SpringBoot(五) 健康检查(上)](https://zhouj000.github.io/2020/01/28/springboot-5/)  
[SpringBoot(六) 健康检查(下)](https://zhouj000.github.io/2020/02/19/springboot-6/)  




> Spring Boot 简化了基于 Spring 的应用开发，通过少量的代码就能创建一个独立的、产品级别的 Spring 应用。 Spring Boot 为 Spring 平台及第三方库提供开箱即用的设置，这样你就可以有条不紊地开始。Spring Boot 的核心思想就是约定大于配置，多数 Spring Boot 应用只需要很少的 Spring 配置。采用 Spring Boot 可以大大的简化你的开发模式，所有你想集成的常用框架，它都有对应的组件支持。

用过springboot后，就会觉得用springboot来开发项目实在是太方便快捷了，它可以无配置集成，极大提高了开发效率


# 准备

首先使用maven引入，spring-boot-starter-parent提供了一些默认的配置与插件
```
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.0.2.RELEASE</version>
	<!--<version>1.5.13.RELEASE</version>-->
</parent>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<!--<artifactId>spring-boot-starter</artifactId>-->
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
然后创建入口类与注解，就可以run了
```
@SpringBootApplication(scanBasePackages = {"com.test"})
public class Application {
	public static void main(String[] args) throws Exception {
        SpringApplication.run(Application.class, args);
    }
}
```



# 启动分析

1 . 创建SpringApplication
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		// 判断WebApplicationType，这里返回SERVLET
		this.webApplicationType = deduceWebApplicationType();
		// 这里有2步，
		// a. 读取META-INF/spring.factories下配置，获取key为ApplicationContextInitializer的实例并实例化
		// 设置初始化，将实例放入initializers列表中
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
		// 设置监听器，和上面的一样，获取key为ApplicationListener的实例并初始化，然后都放到listeners列表中
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		// b. 推断应用入口类
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```
![webapplicationType](/img/in-post/2018/10/webapplicationType.png)

1-a . getSpringFactoriesInstances
```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
		Class<?>[] parameterTypes, Object... args) {
	ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
	// Use names and ensure unique to protect against duplicates
	// ----> 用set保存名称且避免重复，key为ApplicationContextInitializer
	Set<String> names = new LinkedHashSet<>(
			SpringFactoriesLoader.loadFactoryNames(type, classLoader));
	// 根据names来进行实例化 --> 反射
	List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
			classLoader, args, names);
	AnnotationAwareOrderComparator.sort(instances);
	return instances;
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
	MultiValueMap<String, String> result = cache.get(classLoader);
	if (result != null) {
		return result;
	}

	try {	// 根据classLoader，读取"META-INF/spring.factories"下所有路径(或从SystemClassLoader获取)
		Enumeration<URL> urls = (classLoader != null ?
				classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
				ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
		result = new LinkedMultiValueMap<>();
		while (urls.hasMoreElements()) {
			URL url = urls.nextElement();
			UrlResource resource = new UrlResource(url);
			// 读取配置
			Properties properties = PropertiesLoaderUtils.loadProperties(resource);
			for (Map.Entry<?, ?> entry : properties.entrySet()) {
				List<String> factoryClassNames = Arrays.asList(
						StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
				result.addAll((String) entry.getKey(), factoryClassNames);
			}
		}
		// 放入缓存，避免重复读取
		cache.put(classLoader, result);
		return result;
	}
	catch (IOException ex) {
		throw new IllegalArgumentException("Unable to load factories from location [" +
				FACTORIES_RESOURCE_LOCATION + "]", ex);
	}
}
```
1-b . 推断应用入口类
```java
private Class<?> deduceMainApplicationClass() {
	try {
		StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
		// 通过异常栈中方法名为main的栈帧来得到入口类的名字
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
```

2 . 执行run方法
```java
public ConfigurableApplicationContext run(String... args) {
	StopWatch stopWatch = new StopWatch();
	stopWatch.start();
	ConfigurableApplicationContext context = null;
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
	// 设置java.awt.headless参数
	this.configureHeadlessProperty();
	// 获取SpringApplicationRunListeners，依然是通过getSpringFactoriesInstances方法获取key为SpringApplicationRunListener的实例
	// EventPublishingRunListener: 利用一个内部的ApplicationEventMulticaster在上下文实际被刷新之前对事件进行处理
	SpringApplicationRunListeners listeners = this.getRunListeners(args);
	// 发布starting事件给listeners
	listeners.starting();

	Collection exceptionReporters;
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
		// ----> a. 准备环境
		ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
		// 设置spring.beaninfo.ignore属性
		this.configureIgnoreBeanInfo(environment);
		// 打印启动时的banner
		Banner printedBanner = this.printBanner(environment);
		// ----> b. 创建ConfigurableApplicationContext，根据之前的webApplicationType类型创建
		context = this.createApplicationContext();
		// 准备异常报告器，类似的通过getSpringFactoriesInstances方法获取key为SpringBootExceptionReporter的实例
		exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, new Object[]{context});
		// ----> c. 准备上下文
		this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
		// ----> d. 上下文刷新 --> AbstractApplicationContext的refresh方法
		this.refreshContext(context);
		// 上下文后置处理，空方法，留给子类扩展
		this.afterRefresh(context, applicationArguments);
		stopWatch.stop();
		if(this.logStartupInfo) {
			(new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
		}
		// 给listeners发出started事件
		listeners.started(context);
		// ----> e. 调用runners
		this.callRunners(context, applicationArguments);
	} catch (Throwable var10) {
		this.handleRunFailure(context, var10, exceptionReporters, listeners);
		throw new IllegalStateException(var10);
	}

	try {
		// 发出running事件
		listeners.running(context);
		return context;
	} catch (Throwable var9) {
		this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
		throw new IllegalStateException(var9);
	}
}
```

2-a . 准备环境
```java
private ConfigurableEnvironment prepareEnvironment(
		SpringApplicationRunListeners listeners,
		ApplicationArguments applicationArguments) {
	// Create and configure the environment
	ConfigurableEnvironment environment = getOrCreateEnvironment();
	// 配置Property Sources与Profiles
	configureEnvironment(environment, applicationArguments.getSourceArgs());
	// 给linsteners发送environmentPrepared事件，其中会把application配置文件载入
	// ConfigFileApplicationListener
	listeners.environmentPrepared(environment);
	bindToSpringApplication(environment);
	if (this.webApplicationType == WebApplicationType.NONE) {
		environment = new EnvironmentConverter(getClassLoader())
				.convertToStandardEnvironmentIfNecessary(environment);
	}
	ConfigurationPropertySources.attach(environment);
	return environment;
}

protected void configureEnvironment(ConfigurableEnvironment environment,
		String[] args) {
	configurePropertySources(environment, args);
	configureProfiles(environment, args);
}
```
![loadApplicationFile](/img/in-post/2018/10/loadApplicationFile.png)

2-b . 创建ConfigurableApplicationContext上下文，这里是SERVLET创建AnnotationConfigServletWebServerApplicationContext
```java
protected ConfigurableApplicationContext createApplicationContext() {
	Class<?> contextClass = this.applicationContextClass;
	if (contextClass == null) {
		try {
			switch (this.webApplicationType) {
			case SERVLET:		// AnnotationConfigServletWebServerApplicationContext
				contextClass = Class.forName(DEFAULT_WEB_CONTEXT_CLASS);
				break;
			case REACTIVE:		// AnnotationConfigReactiveWebServerApplicationContext
				contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
				break;
			default:			// AnnotationConfigApplicationContext
				contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
			}
		}
		catch (ClassNotFoundException ex) {
			throw new IllegalStateException(
					"Unable create a default ApplicationContext, "
							+ "please specify an ApplicationContextClass",
					ex);
		}
	}
	return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```
![annotationConfigServletWebServerApplicationContext](/img/in-post/2018/10/annotationConfigServletWebServerApplicationContext.png)
[AnnotationConfigApplicationContext非web环境下的启动容器](https://blog.csdn.net/jamet/article/details/77417997)  
[AnnotationConfigEmbeddedWebApplicationContext默认web环境下的启动容器](https://blog.csdn.net/jamet/article/details/77428946)

2-c . 准备上下文
```java
private void prepareContext(ConfigurableApplicationContext context,
		ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
		ApplicationArguments applicationArguments, Banner printedBanner) {
	context.setEnvironment(environment);
	// 配置beanNameGenerator以及resourceLoader
	postProcessApplicationContext(context);
	// 调用所有初始化器进行初始化
	applyInitializers(context);
	// 向listeners发布contextPrepared事件(EventPublishingRunListener为空方法处理)
	listeners.contextPrepared(context);
	if (this.logStartupInfo) {
		logStartupInfo(context.getParent() == null);
		logStartupProfileInfo(context);
	}

	// Add boot specific singleton beans
	// 添加特殊单例bean：springApplicationArguments与springBootBanner
	context.getBeanFactory().registerSingleton("springApplicationArguments",
			applicationArguments);
	if (printedBanner != null) {
		context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
	}

	// 加载sources		Load the sources
	Set<Object> sources = getAllSources();
	Assert.notEmpty(sources, "Sources must not be empty");
	// 创建BeanDefinitionLoader，加载bean到上下文
	load(context, sources.toArray(new Object[0]));
	// 触发contextLoaded事件给listeners
	listeners.contextLoaded(context);
}
```

2-d . 上下文刷新
```java
private void refreshContext(ConfigurableApplicationContext context) {
	// 刷新上下文
	refresh(context);
	if (this.registerShutdownHook) {
		try {
			// 注册ShutdownHook
			context.registerShutdownHook();
		}
		catch (AccessControlException ex) {
			// Not allowed in some environments.
		}
	}
}

protected void refresh(ApplicationContext applicationContext) {
	Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
	// ServletWebServerApplicationContext.refresh --> AbstractApplicationContext.refresh
	// 同样由于继承自GenericApplicationContext所以refreshBeanFactory不会注册BeanDefinitions
	// 会在invokeBeanFactoryPostProcessors里获取ConfigurationClassPostProcessor，执行其postProcessBeanDefinitionRegistry方法去注册
	((AbstractApplicationContext) applicationContext).refresh();
}
```

2-e . 调用runners
```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {
	List<Object> runners = new ArrayList<>();
	runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
	runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
	AnnotationAwareOrderComparator.sort(runners);
	for (Object runner : new LinkedHashSet<>(runners)) {
		if (runner instanceof ApplicationRunner) {
			// 调用run方法
			callRunner((ApplicationRunner) runner, args);
		}
		if (runner instanceof CommandLineRunner) {
			// 调用run方法
			callRunner((CommandLineRunner) runner, args);
		}
	}
}
```



# auto-configuration

@SpringBootApplication注解是一个组合注解，需要注意的有@SpringBootConfiguration、@EnableAutoConfiguration、@ComponentScan这三个注解
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration		// 组合了@Configuration
@EnableAutoConfiguration
@ComponentScan(					// 代替了<context:component-scan>
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {}
```

接着查看@EnableAutoConfiguration注解，发现导入了AutoConfigurationImportSelector类，@AutoConfigurationPackage注解里又导入了Registrar类
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {}

...
@Import({Registrar.class})
public @interface AutoConfigurationPackage{}
```

ConfigurationClassPostProcessor在初始化时会调用postProcessBeanDefinitionRegistry方法和postProcessBeanFactory方法，通过判断factoriesPostProcessed调用processConfigBeanDefinitions方法，然后一直调用到AutoConfigurationImportSelector的selectImports方法。  
那就打开AutoConfigurationImportSelector类看一下，实现了ImportSelector接口，作用与注解 @Import类似。其中核心方法selectImports，根据 importingClassMetadata的值，从带有注解@Configuration的类中选择并返回合适的类名数组，将其导入Spring容器。
```java
// AutoConfigurationImportSelector
public String[] selectImports(AnnotationMetadata annotationMetadata) {
	if(!this.isEnabled(annotationMetadata)) {
		return NO_IMPORTS;
	} else {
		// 获取自动配置的信息  META-INF/spring-autoconfigure-metadata.properties
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
		// 获取入口类上注解@EnableAutoConfiguration的配置信息
		AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
		// ----> 获取候选自动配置类名
		List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
		// 去重，使用了LinkedHashSet
		configurations = this.removeDuplicates(configurations);
		// 获取排除项
		Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
		this.checkExcludedClasses(configurations, exclusions);
		// 去除排除项
		configurations.removeAll(exclusions);
		// ----> 过滤
		configurations = this.filter(configurations, autoConfigurationMetadata);
		// 获取listeners(ConditionEvaluationReportAutoConfigurationImportListener)，创建AutoConfigurationImportEvent事件，遍历调用onAutoConfigurationImportEvent方法(用于记录)
		this.fireAutoConfigurationImportEvents(configurations, exclusions);
		return StringUtils.toStringArray(configurations);
	}
}

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
	// 同样通过SpringFactoriesLoader从配置文件"META-INF/spring.factories"中加载配置，key为EnableAutoConfiguration的
	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
	Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
	return configurations;
}

private List<String> filter(List<String> configurations,
		AutoConfigurationMetadata autoConfigurationMetadata) {
	long startTime = System.nanoTime();
	String[] candidates = StringUtils.toStringArray(configurations);
	boolean[] skip = new boolean[candidates.length];
	boolean skipped = false;
	// 获取filter: OnClassCondition
	for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
		// 判断设入beanClassLoader、beanFactory、environment、resourceLoader
		invokeAwareMethods(filter);
		// ----> 判断是否匹配
		boolean[] match = filter.match(candidates, autoConfigurationMetadata);
		for (int i = 0; i < match.length; i++) {
			if (!match[i]) {
				skip[i] = true;
				skipped = true;
			}
		}
	}
	// 没有要跳过的
	if (!skipped) {
		return configurations;
	}
	List<String> result = new ArrayList<>(candidates.length);
	for (int i = 0; i < candidates.length; i++) {
		if (!skip[i]) {
			// 符合条件的加入
			result.add(candidates[i]);
		}
	}
	if (logger.isTraceEnabled()) {
		int numberFiltered = configurations.size() - result.size();
		logger.trace("Filtered " + numberFiltered + " auto configuration class in "
				+ TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)
				+ " ms");
	}
	return new ArrayList<>(result);
}

// OnClassCondition
public boolean[] match(String[] autoConfigurationClasses,
		AutoConfigurationMetadata autoConfigurationMetadata) {
	ConditionEvaluationReport report = getConditionEvaluationReport();
	// ---->
	ConditionOutcome[] outcomes = getOutcomes(autoConfigurationClasses,
			autoConfigurationMetadata);
	boolean[] match = new boolean[outcomes.length];
	for (int i = 0; i < outcomes.length; i++) {
		match[i] = (outcomes[i] == null || outcomes[i].isMatch());
		if (!match[i] && outcomes[i] != null) {
			logOutcome(autoConfigurationClasses[i], outcomes[i]);
			if (report != null) {
				report.recordConditionEvaluation(autoConfigurationClasses[i], this,
						outcomes[i]);
			}
		}
	}
	return match;
}
```

得到这些后，会将对应的配置项当做标注了@configuration的javaConfig形式的IOC容器配置类
```java
// ConfigurationClassParser
for (DeferredImportSelectorGrouping grouping : groupings.values()) {
	grouping.getImports().forEach((entry) -> {
		ConfigurationClass configurationClass = configurationClasses.get(
				entry.getMetadata());
		try {
			// ----> 
			processImports(configurationClass, asSourceClass(configurationClass),
					asSourceClasses(entry.getImportClassName()), false);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(
					"Failed to process import candidates for configuration class [" +
							configurationClass.getMetadata().getClassName() + "]", ex);
		}
	});
}

private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
		Collection<SourceClass> importCandidates, boolean checkForCircularImports) {
	if (importCandidates.isEmpty()) {
		return;
	}
	if (checkForCircularImports && isChainedImportOnStack(configClass)) {
		this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
	}
	else {
		this.importStack.push(configClass);
		try {
			for (SourceClass candidate : importCandidates) {
				if (candidate.isAssignable(ImportSelector.class)) {
					// Candidate class is an ImportSelector -> delegate to it to determine imports
					Class<?> candidateClass = candidate.loadClass();
					ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
					ParserStrategyUtils.invokeAwareMethods(
							selector, this.environment, this.resourceLoader, this.registry);
					if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
						this.deferredImportSelectors.add(
								new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
					}
					else {
						String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
						Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
						processImports(configClass, currentSourceClass, importSourceClasses, false);
					}
				}
				else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
					// Candidate class is an ImportBeanDefinitionRegistrar ->
					// delegate to it to register additional bean definitions
					Class<?> candidateClass = candidate.loadClass();
					ImportBeanDefinitionRegistrar registrar =
							BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
					ParserStrategyUtils.invokeAwareMethods(
							registrar, this.environment, this.resourceLoader, this.registry);
					configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
				}
				else {
					// Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
					// process it as an @Configuration class
					// 注册Configuration类
					this.importStack.registerImport(
							currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
					// ---->
					processConfigurationClass(candidate.asConfigClass(configClass));
				}
			}
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(
					"Failed to process import candidates for configuration class [" +
					configClass.getMetadata().getClassName() + "]", ex);
		}
		finally {
			this.importStack.pop();
		}
	}
}

protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
	// 调用ConditionEvaluator.shouldSkip判断
	if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
		return;
	}

	ConfigurationClass existingClass = this.configurationClasses.get(configClass);
	if (existingClass != null) {
		if (configClass.isImported()) {
			if (existingClass.isImported()) {
				existingClass.mergeImportedBy(configClass);
			}
			// Otherwise ignore new imported config class; existing non-imported class overrides it.
			return;
		}
		else {
			// Explicit bean definition found, probably replacing an import.
			// Let's remove the old one and go with the new one.
			this.configurationClasses.remove(configClass);
			this.knownSuperclasses.values().removeIf(configClass::equals);
		}
	}

	// Recursively process the configuration class and its superclass hierarchy.
	SourceClass sourceClass = asSourceClass(configClass);
	do {
		// ----> 解析Configuration注解
		sourceClass = doProcessConfigurationClass(configClass, sourceClass);
	}
	while (sourceClass != null);

	this.configurationClasses.put(configClass, configClass);
}

protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
		throws IOException {

	// Recursively process any member (nested) classes first
	processMemberClasses(configClass, sourceClass);

	// Process any @PropertySource annotations
	for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), PropertySources.class,
			org.springframework.context.annotation.PropertySource.class)) {
		if (this.environment instanceof ConfigurableEnvironment) {
			processPropertySource(propertySource);
		}
		else {
			logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
					"]. Reason: Environment must implement ConfigurableEnvironment");
		}
	}

	// Process any @ComponentScan annotations
	Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
	if (!componentScans.isEmpty() &&
			!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
		for (AnnotationAttributes componentScan : componentScans) {
			// The config class is annotated with @ComponentScan -> perform the scan immediately
			Set<BeanDefinitionHolder> scannedBeanDefinitions =
					this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
			// Check the set of scanned definitions for any further config classes and parse recursively if needed
			for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
				BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
				if (bdCand == null) {
					bdCand = holder.getBeanDefinition();
				}
				if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
					parse(bdCand.getBeanClassName(), holder.getBeanName());
				}
			}
		}
	}

	// Process any @Import annotations
	processImports(configClass, sourceClass, getImports(sourceClass), true);

	// Process any @ImportResource annotations
	AnnotationAttributes importResource =
			AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
	if (importResource != null) {
		String[] resources = importResource.getStringArray("locations");
		Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
		for (String resource : resources) {
			String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
			configClass.addImportedResource(resolvedResource, readerClass);
		}
	}

	// Process individual @Bean methods
	Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
	for (MethodMetadata methodMetadata : beanMethods) {
		configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
	}

	// Process default methods on interfaces
	processInterfaces(configClass, sourceClass);

	// Process superclass, if any
	if (sourceClass.getMetadata().hasSuperClass()) {
		String superclass = sourceClass.getMetadata().getSuperClassName();
		if (superclass != null && !superclass.startsWith("java") &&
				!this.knownSuperclasses.containsKey(superclass)) {
			this.knownSuperclasses.put(superclass, configClass);
			// Superclass found, return its annotation metadata and recurse
			return sourceClass.getSuperClass();
		}
	}

	// No superclass -> processing is complete
	return null;
}
```

## 条件注解
```java
@ConditionalOnClass
@ConditionalOnBean
@ConditionalOnProperty
@ConditionalOnExpression // 等等
```
比如查看@ConditionalOnClass注解
```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
// OnClassCondition.match方法就是上面filter里执行的方法
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {}
```
SpringBoot使用ConditionEvaluator这个内部类完成条件注解的解析和判断。在Spring容器的refresh过程中，只有跟解析或者注册bean有关系的类都会使用ConditionEvaluator完成条件注解的判断，这个过程中一些类不满足条件的话就会被skip
```java
// ConfigurationClassParser
this.conditionEvaluator = new ConditionEvaluator(registry, environment, resourceLoader);

// ConditionEvaluator
public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
	// 如果这个类没有被@Conditional注解所修饰，不会skip
	if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
		return false;
	}
	// 如果参数中沒有设置条件注解的生效阶段
	if (phase == null) {
		// 是配置类的话直接使用PARSE_CONFIGURATION阶段
		if (metadata instanceof AnnotationMetadata &&
				ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
			return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
		}
		// 否则使用REGISTER_BEAN阶段
		return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
	}
	// 要解析的配置类的条件集合
	List<Condition> conditions = new ArrayList<>();
	// 获取配置类的条件注解得到条件数据，并添加到集合中
	for (String[] conditionClasses : getConditionClasses(metadata)) {
		for (String conditionClass : conditionClasses) {
			Condition condition = getCondition(conditionClass, this.context.getClassLoader());
			conditions.add(condition);
		}
	}
	// 对条件集合排序
	AnnotationAwareOrderComparator.sort(conditions);
	// 遍历条件集合
	for (Condition condition : conditions) {
		ConfigurationPhase requiredPhase = null;
		if (condition instanceof ConfigurationCondition) {
			requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
		}
		// 阶段一致切不满足条件的话，返回true并跳过这个bean的解析
		if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
			return true;
		}
	}
	return false;
}
```

# 举例

可以看出，自动创建了非常熟悉的DispatcherServlet与MultipartResolver类
```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
@AutoConfigureAfter(ServletWebServerFactoryAutoConfiguration.class)
@EnableConfigurationProperties(ServerProperties.class)
public class DispatcherServletAutoConfiguration {

	// ...

	@Configuration
	@Conditional(DefaultDispatcherServletCondition.class)
	@ConditionalOnClass(ServletRegistration.class)
	@EnableConfigurationProperties(WebMvcProperties.class)
	protected static class DispatcherServletConfiguration {

		private final WebMvcProperties webMvcProperties;

		private final ServerProperties serverProperties;

		public DispatcherServletConfiguration(WebMvcProperties webMvcProperties,
				ServerProperties serverProperties) {
			this.webMvcProperties = webMvcProperties;
			this.serverProperties = serverProperties;
		}

		@Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public DispatcherServlet dispatcherServlet() {
			DispatcherServlet dispatcherServlet = new DispatcherServlet();
			dispatcherServlet.setDispatchOptionsRequest(
					this.webMvcProperties.isDispatchOptionsRequest());
			dispatcherServlet.setDispatchTraceRequest(
					this.webMvcProperties.isDispatchTraceRequest());
			dispatcherServlet.setThrowExceptionIfNoHandlerFound(
					this.webMvcProperties.isThrowExceptionIfNoHandlerFound());
			return dispatcherServlet;
		}

		@Bean
		@ConditionalOnBean(MultipartResolver.class)
		@ConditionalOnMissingBean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
		public MultipartResolver multipartResolver(MultipartResolver resolver) {
			// Detect if the user has created a MultipartResolver but named it incorrectly
			return resolver;
		}
		
		// ...
	}
	
	// ...
}
```
使用@EnableConfigurationProperties、@ConfigurationProperties提供配置参数，
```java
@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {
	// ...
}
```
刷新中applyBeanPostProcessorsBeforeInitialization方法会触发ConfigurationPropertiesBindingPostProcessor.postProcessBeforeInitialization方法
```java
public Object postProcessBeforeInitialization(Object bean, String beanName)
		throws BeansException {
	// 获取类上的@ConfigurationProperties注解
	ConfigurationProperties annotation = getAnnotation(bean, beanName,
			ConfigurationProperties.class);
	if (annotation != null) {
		bind(bean, beanName, annotation);
	}
	return bean;
}
```



扩展：  
[为什么说 Java 程序员到了必须掌握 Spring Boot 的时候？](http://www.ityouknow.com/springboot/2018/06/12/spring-boo-java-simple.html)  